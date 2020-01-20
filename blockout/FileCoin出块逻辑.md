# filecoin出块逻辑整理

## 前言

   filecoin的出块功能是矿工模块提供的,本文简要总结一下filecoin的相关出块逻辑.

## filecoin出块函数调用
![Image text](./20170518152848.png)

## 矿工模块的相关功能

### Start

```go
func (s *timingScheduler) Start(miningCtx context.Context) (<-chan Output, *sync.WaitGroup) {
	var doneWg sync.WaitGroup
	outCh := make(chan Output, 1)

	// loop mining work
	doneWg.Add(1)
	go func() {
		s.mineLoop(miningCtx, outCh, &doneWg)
		doneWg.Done()
	}()
	s.isStarted = true

	return outCh, &doneWg
}
```

start函数是矿工模块启动的函数,该函数创建一个outCh的管道,并启动一个协程运行mineLoop函数.

### mineLoop

```go
func (s *timingScheduler) mineLoop(miningCtx context.Context, outCh chan Output, doneWg *sync.WaitGroup) {
	workContext, workCancel := context.WithCancel(miningCtx) // nolint:staticcheck
	for {
		currEpoch := s.chainClock.WaitNextEpoch(miningCtx)
		select { // check for interrupt during waiting
		case <-miningCtx.Done():
			s.isStarted = false
			return // nolint:govet
		default:
		}
		workCancel() // cancel any late work from last epoch

		// check if we are skipping and don't mine if so
		if s.isSkipping() {
			continue
		}

		workContext, workCancel = context.WithCancel(miningCtx) // nolint: govet
		base, err := s.pollHeadFunc()
		if err != nil {
			log.Errorf("error polling head from mining scheduler %s", err)
		}
		h, err := base.Height()
		if err != nil {
			log.Errorf("error getting height from base", err)
		}
		nullBlkCount := uint64(currEpoch.AsBigInt().Int64()) - h - 1
		doneWg.Add(1)
		go func(ctx context.Context) {
			s.worker.Mine(ctx, base, nullBlkCount, outCh)
			doneWg.Done()
		}(workContext)
	}
}
```

mineloop 函数的核心是一个循环,循环的终止条件是上层context调用Done函数.
+ `base, err := s.pollHeadFunc()` 获取当前区块头
+ `nullBlkCount := uint64(currEpoch.AsBigInt().Int64()) - h - 1` 计算需要生产空块的个数
+ `s.worker.Mine(ctx, base, nullBlkCount, outCh)`通过Mine函数生产区块

### Mine

```go
func (w *DefaultWorker) Mine(ctx context.Context, base block.TipSet, nullBlkCount uint64, outCh chan<- Output) (won bool) {
   log.Info("Worker.Mine")
   if !base.Defined() {
      log.Warn("Worker.Mine returning because it can't mine on an empty tipset")
      outCh <- Output{Err: errors.New("bad input tipset with no blocks sent to Mine()")}
      return
   }

   log.Debugf("Mining on tipset: %s, with %d null blocks.", base.String(), nullBlkCount)
   if ctx.Err() != nil {
      log.Warnf("Worker.Mine returning with ctx error %s", ctx.Err().Error())
      return
   }

   // Read uncached worker address
   workerAddr, err := w.api.MinerGetWorkerAddress(ctx, w.minerAddr, base.Key())
   if err != nil {
      outCh <- Output{Err: err}
      return
   }

   // lookback consensus.ElectionLookback
   prevTicket, err := base.MinTicket()
   if err != nil {
      log.Warnf("Worker.Mine couldn't read parent ticket %s", err)
      outCh <- Output{Err: err}
      return
   }

   nextTicket, err := w.ticketGen.NextTicket(prevTicket, workerAddr, w.workerSigner)
   if err != nil {
      log.Warnf("Worker.Mine couldn't generate next ticket %s", err)
      outCh <- Output{Err: err}
      return
   }

   // Provably delay for the blocktime
   done := make(chan struct{})
   go func() {
      defer close(done)
      // TODO #3703 actually launch election post work here
      done <- struct{}{}
   }()

   select {
   case <-ctx.Done():
      log.Infow("Mining run on tipset with null blocks canceled.", "tipset", base, "nullBlocks", nullBlkCount)
      return
   case <-done:
   }

   // lookback consensus.ElectionLookback for the election ticket
   baseHeight, err := base.Height()
   if err != nil {
      log.Warnf("Worker.Mine couldn't read base height %s", err)
      outCh <- Output{Err: err}
      return
   }
   ancestors, err := w.getAncestors(ctx, base, types.NewBlockHeight(baseHeight+nullBlkCount+1))
   if err != nil {
      log.Warnf("Worker.Mine couldn't get ancestorst %s", err)
      outCh <- Output{Err: err}
      return
   }
   electionTicket, err := sampling.SampleNthTicket(consensus.ElectionLookback-1, ancestors)
   if err != nil {
      log.Warnf("Worker.Mine couldn't read parent ticket %s", err)
      outCh <- Output{Err: err}
      return
   }

   // Run an election to check if this miner has won the right to mine
   electionProof, err := w.election.DeprecatedRunElection(electionTicket, workerAddr, w.workerSigner, nullBlkCount)
   if err != nil {
      log.Errorf("failed to run local election: %s", err)
      outCh <- Output{Err: err}
      return
   }
   powerTable, err := w.getPowerTable(ctx, base.Key())
   if err != nil {
      log.Errorf("Worker.Mine couldn't get snapshot for tipset: %s", err.Error())
      outCh <- Output{Err: err}
      return
   }
   weHaveAWinner, err := w.election.DeprecatedIsElectionWinner(ctx, powerTable, electionTicket, nullBlkCount, electionProof, workerAddr, w.minerAddr)
   if err != nil {
      log.Errorf("Worker.Mine couldn't run election: %s", err.Error())
      outCh <- Output{Err: err}
      return
   }

   // This address has mining rights, so mine a block
   if weHaveAWinner {
      next, err := w.Generate(ctx, base, nextTicket, electionProof, nullBlkCount)
      if err == nil {
         log.Debugf("Worker.Mine generates new winning block! %s", next.Cid().String())
      }
      outCh <- NewOutput(next, err)
      won = true
      return
   }
   return
}
```

+ `workerAddr, err := w.api.MinerGetWorkerAddress(ctx, w.minerAddr, base.Key())`     获取当前工作区的地址
+ `baseHeight, err := base.Height()`			获取当前块高度
+ `ancestors, err := w.getAncestors(ctx, base, types.NewBlockHeight(baseHeight+nullBlkCount+1))` 获取所有区块的TipSet
+ `electionTicket, err := sampling.SampleNthTicket(consensus.ElectionLookback-1, ancestors)` 通过相关的算法计算是否拥有出块的权限,这个类似BTC的挖矿,通过计算出一个值来获取出块的权限
+ `electionProof, err := w.election.DeprecatedRunElection(electionTicket, workerAddr, w.workerSigner, nullBlkCount)` 验证当前是否拥有出块权限
+ `next, err := w.Generate(ctx, base, nextTicket, electionProof, nullBlkCount)` 如果拥有出块权限,另起协程进行区块生产
+ `outCh <- NewOutput(next, err)` 生产完区块输出到outCh中

### Generate

```go
// Generate returns a new block created from the messages in the pool.
func (w *DefaultWorker) Generate(ctx context.Context,
   baseTipSet block.TipSet,
   ticket block.Ticket,
   electionProof block.VRFPi,
   nullBlockCount uint64) (*block.Block, error) {

   generateTimer := time.Now()
   defer func() {
      log.Infof("[TIMER] DefaultWorker.Generate baseTipset: %s - elapsed time: %s", baseTipSet.String(), time.Since(generateTimer).Round(time.Millisecond))
   }()

   stateTree, err := w.getStateTree(ctx, baseTipSet.Key())
   if err != nil {
      return nil, errors.Wrap(err, "get state tree")
   }

   powerTable, err := w.getPowerTable(ctx, baseTipSet.Key())
   if err != nil {
      return nil, errors.Wrap(err, "get power table")
   }

   hasPower, err := powerTable.HasPower(ctx, w.minerAddr)
   if err != nil {
      return nil, err
   }
   if !hasPower {
      return nil, errors.Errorf("bad miner address, miner must store files before mining: %s", w.minerAddr)
   }

   weight, err := w.getWeight(ctx, baseTipSet)
   if err != nil {
      return nil, errors.Wrap(err, "get weight")
   }

   baseHeight, err := baseTipSet.Height()
   if err != nil {
      return nil, errors.Wrap(err, "get base tip set height")
   }

   blockHeight := baseHeight + nullBlockCount + 1

   ancestors, err := w.getAncestors(ctx, baseTipSet, types.NewBlockHeight(blockHeight))
   if err != nil {
      return nil, errors.Wrap(err, "get base tip set ancestors")
   }

   // Construct list of message candidates for inclusion.
   // These messages will be processed, and those that fail excluded from the block.
   pending := w.messageSource.Pending()
   mq := NewMessageQueue(pending)
   candidateMsgs := orderMessageCandidates(mq.Drain())

   // run state transition to learn which messages are valid
   vms := vm.NewStorageMap(w.blockstore)
   results, err := w.processor.ApplyMessagesAndPayRewards(ctx, stateTree, vms, types.UnwrapSigned(candidateMsgs),
      w.minerOwnerAddr, types.NewBlockHeight(blockHeight), ancestors)
   if err != nil {
      return nil, errors.Wrap(err, "generate apply messages")
   }

   var blsAccepted []*types.SignedMessage
   var secpAccepted []*types.SignedMessage

   // Align the results with the candidate signed messages to accumulate the messages lists
   // to include in the block, and handle failed messages.
   for i, r := range results {
      msg := candidateMsgs[i]
      if r.Failure == nil {
         if msg.Message.From.Protocol() == address.BLS {
            blsAccepted = append(blsAccepted, msg)
         } else {
            secpAccepted = append(secpAccepted, msg)
         }
      } else if r.FailureIsPermanent {
         // Remove message that can never succeed from the message pool now.
         // There might be better places to do this, such as wherever successful messages are removed
         // from the pool, or by posting the failure to an event bus to be handled async.
         log.Infof("permanent ApplyMessage failure, [%s] (%s)", msg, r.Failure)
         mc, err := msg.Cid()
         if err == nil {
            w.messageSource.Remove(mc)
         } else {
            log.Warnf("failed to get CID from message", err)
         }
      } else {
         // This message might succeed in the future, so leave it in the pool for now.
         log.Infof("temporary ApplyMessage failure, [%s] (%s)", msg, r.Failure)
      }
   }

   // Create an aggregage signature for messages
   unwrappedBLSMessages, blsAggregateSig, err := aggregateBLS(blsAccepted)
   if err != nil {
      return nil, errors.Wrap(err, "could not aggregate bls messages")
   }

   // Persist messages to ipld storage
   txMeta, err := w.messageStore.StoreMessages(ctx, secpAccepted, unwrappedBLSMessages)
   if err != nil {
      return nil, errors.Wrap(err, "error persisting messages")
   }

   // get tipset state root and receipt root
   baseStateRoot, err := w.tsMetadata.GetTipSetStateRoot(baseTipSet.Key())
   if err != nil {
      return nil, errors.Wrapf(err, "error retrieving state root for tipset %s", baseTipSet.Key().String())
   }

   baseReceiptRoot, err := w.tsMetadata.GetTipSetReceiptsRoot(baseTipSet.Key())
   if err != nil {
      return nil, errors.Wrapf(err, "error retrieving receipt root for tipset %s", baseTipSet.Key().String())
   }

   now := w.clock.Now()
   next := &block.Block{
      Miner:                   w.minerAddr,
      Height:                  types.Uint64(blockHeight),
      Messages:                txMeta,
      MessageReceipts:         baseReceiptRoot,
      Parents:                 baseTipSet.Key(),
      ParentWeight:            types.Uint64(weight),
      DeprecatedElectionProof: electionProof,
      StateRoot:               baseStateRoot,
      Ticket:                  ticket,
      Timestamp:               types.Uint64(now.Unix()),
      BLSAggregateSig:         blsAggregateSig,
   }

   workerAddr, err := w.api.MinerGetWorkerAddress(ctx, w.minerAddr, baseTipSet.Key())
   if err != nil {
      return nil, errors.Wrap(err, "failed to read workerAddr during block generation")
   }
   next.BlockSig, err = w.workerSigner.SignBytes(next.SignatureData(), workerAddr)
   if err != nil {
      return nil, errors.Wrap(err, "failed to sign block")
   }

   return next, nil
}
```

+ stateTree, err := w.getStateTree(ctx, baseTipSet.Key()) 获取stateTree
+ powerTable, err := w.getPowerTable(ctx, baseTipSet.Key()) 获取powertable
+ hasPower, err := powerTable.HasPower(ctx, w.minerAddr)  获取hash
+ blockHeight := baseHeight + nullBlockCount + 1  获取区块高度
+ results, err := w.processor.ApplyMessagesAndPayRewards(ctx, stateTree, vms, types.UnwrapSigned(candidateMsgs), 区块里面应该处理所有的交易,并派发佣金 接下来的循环就是判断一下成功的交易和不成共的交易,并且生成message.
+ baseStateRoot, err := w.tsMetadata.GetTipSetStateRoot(baseTipSet.Key()) baseReceiptRoot, err := w.tsMetadata.GetTipSetReceiptsRoot(baseTipSet.Key()) 生成stateroot和receiptroot.然后构建区块next
+ next.BlockSig, err = w.workerSigner.SignBytes(next.SignatureData(), workerAddr)  对区块进行签名

以上就是filecoin生产区块的逻辑,但是区块生产以后怎么处理呢?在Mine函数中`outCh <- NewOutput(next, err)` 生产完区块输出到outCh中,但是输出到outch以后呢?接下来看一下mine启动的地方,在`func (node *Node) StartMining(ctx context.Context) error ` 函数中有启动的语句`outCh, doneWg := node.BlockMining.MiningScheduler.Start(miningCtx)`,在这个语句后面有对outCh的处理`go node.handleNewMiningOutput(miningCtx, outCh)`

### handleNewMiningOutput

```go
func (node *Node) handleNewMiningOutput(ctx context.Context, miningOutCh <-chan mining.Output) {
   defer func() {
      node.BlockMining.MiningDoneWg.Done()
   }()
   for {
      select {
      case <-ctx.Done():
         return
      case output, ok := <-miningOutCh:
         if !ok {
            return
         }
         if output.Err != nil {
            log.Errorf("stopping mining. error: %s", output.Err.Error())
            node.StopMining(context.Background())
         } else {
            node.BlockMining.MiningDoneWg.Add(1)
            go func() {
               if node.IsMining() {
                  node.BlockMining.AddNewlyMinedBlock(ctx, output.NewBlock)
               }
               node.BlockMining.MiningDoneWg.Done()
            }()
         }
      }
   }

}
```

这个函数中接收到新的区块以后会调用`node.BlockMining.AddNewlyMinedBlock(ctx, output.NewBlock)`函数在AddNewlyMinedBlock函数里面会调用AddNewBlock函数

### AddNewBlock

```go
// AddNewBlock receives a newly mined block and stores, validates and propagates it to the network.
func (node *Node) AddNewBlock(ctx context.Context, b *block.Block) (err error) {
   ctx, span := trace.StartSpan(ctx, "Node.AddNewBlock")
   span.AddAttributes(trace.StringAttribute("block", b.Cid().String()))
   defer tracing.AddErrorEndSpan(ctx, span, &err)

   // Put block in storage wired to an exchange so this node and other
   // nodes can fetch it.
   log.Debugf("putting block in bitswap exchange: %s", b.Cid().String())
   blkCid, err := node.Blockstore.CborStore.Put(ctx, b)
   if err != nil {
      return errors.Wrap(err, "could not add new block to online storage")
   }

   log.Debugf("syncing new block: %s", b.Cid().String())
   go func() {
      err = node.syncer.BlockTopic.Publish(ctx, b.ToNode().RawData())
      if err != nil {
         log.Errorf("error publishing new block on block topic %s", err)
      }
   }()
   ci := block.NewChainInfo(node.Host().ID(), node.Host().ID(), block.NewTipSetKey(blkCid), uint64(b.Height))
   return node.syncer.ChainSyncManager.BlockProposer().SendOwnBlock(ci)
}
```
+ blkCid, err := node.Blockstore.CborStore.Put(ctx, b) 将区块存储在存储模块
+ err = node.syncer.BlockTopic.Publish(ctx, b.ToNode().RawData()) 通过libp2p模块将区块广播出去
+ node.syncer.ChainSyncManager.BlockProposer().SendOwnBlock(ci) 将区块通过异步模块添加到链上面

以上就是filecoin整个出块相关逻辑,由于filecoin目前处于密集开发阶段,上面的代码或有些许改动.



