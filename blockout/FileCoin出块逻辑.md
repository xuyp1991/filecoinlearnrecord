# filecoin出块逻辑整理

## 前言

   filecoin的出块功能是矿工模块提供的,本文简要总结一下filecoin的相关出块逻辑.

## filecoin出块函数调用
![Image text](https://github.com/xuyp1991/filecoinlearnrecord/blob/master/blockout/filecoin_blockout.png)

## 出块函数调用说明

根据图片所示,矿工模块启动以后会在MineLoop进行循环出块,Mine函数进行挖矿,获取出块资格,有了出块资格以后调用Generate函数生产区块,获得区块以后签名,最后输出到outCh管道中

StartMining调用handleNewMiningOutput来处理outCh管道中的区块.

## 关于挖矿的关键代码

```go
prevTicket, err := base.MinTicket()
nextTicket, err := w.ticketGen.NextTicket(prevTicket, workerAddr, w.workerSigner)
ancestors, err := w.getAncestors(ctx, base, types.NewBlockHeight(baseHeight+nullBlkCount+1))
electionTicket, err := sampling.SampleNthTicket(consensus.ElectionLookback-1, ancestors)
postRandomness, err := w.election.GeneratePoStRandomness(electionTicket, workerAddr, w.workerSigner, nullBlkCount)
candidates, err := w.election.GenerateCandidates(postRandomness, sortedSectorInfos, w.poster)
sectorNum, err := powerTable.NumSectors(ctx, w.minerAddr)
networkPower, err := powerTable.Total(ctx)
sectorSize, err := powerTable.SectorSize(ctx, w.minerAddr)
for _, candidate := range candidates {
   hasher.Bytes(candidate.PartialTicket)
   challengeTicket := hasher.Hash()
   if w.election.CandidateWins(challengeTicket, w.poster, sectorNum, 0, networkPower.Uint64(), sectorSize.Uint64()) {
      winners = append(winners, candidate)
   }
}
post, err := w.election.GeneratePoSt(sortedSectorInfos, postRandomness, winners, w.poster)
next, err := w.Generate(ctx, base, nextTicket, nullBlkCount, postRandomness, winners, post)
outCh <- NewOutput(next, err)
```

上面代码就是获取出块权的相关代码.

|             操作              |                         含义                         |
| :---------------------------: | :--------------------------------------------------: |
|         getAncestors          | 获得处理所有状态转换所需的祖先链的轮数的区块的TipSet |
|        SampleNthTicket        |                  获取父区块的ticket                  |
|      GenerateCandidates       |                 生产候选者的候选分票                 |
|          NumSectors           |                获取矿工承诺的扇区个数                |
|       powerTable.Total        |              获取所有矿工提供的扇区总数              |
|     powerTable.SectorSize     |                获取当前矿工的扇区个数                |
|         CandidateWins         |              计算当前矿工是否赢得出块权              |
|         GeneratePoSt          |            为赢得出块权的矿工生产出块证明            |
|           Generate            |                       生产区块                       |
| outCh <- NewOutput(next, err) |              将区块放置到outCh 管道中.               |

## 区块生产时的关键代码

### block结构体

```
Block
    |----- Miner                生产者地址
    |----- Ticket               生产者的证明
    |----- Parents              父区块的TipSetKey的集合
    |----- ParentWeight         ParentWeight是父集的总链重。
    |----- Height               当前区块高度
    |----- Messages             消息的集合
    |----- StateRoot            状态树的cid指针
    |----- MessageReceipts      Messages对应的data的地址
    |----- DeprecatedElectionProof  痕迹,证明该矿工获得了出块权
    |----- PoStRandomness       用于生成候选人的可验证随机性证明
    |----- PoStPartialTickets   和区块一起提交的出块权的证明
    |----- PoStSectorIDs        获取出块权的扇区的ID
    |----- PoStChallengeIDXs    获取出块权的挑战指数
    |----- PoStProof            出块权有效的证明
    |----- Timestamp            时间戳
    |----- BlockSig             区块签名
    |----- BLSAggregateSig      区块上所有BLS的聚合签名
    |----- cachedCid            cachecid
    |----- cachedBytes          cachedBytes
```

先给block归下类

基础信息:
+ Miner
+ Parents
+ ParentWeight
+ Height
+ Timestamp
+ BlockSig
+ BLSAggregateSig
+ cachedCid
+ cachedBytes
  
业务信息:
+ Messages
+ StateRoot
+ MessageReceipts

挖矿信息:
+ Ticket
+ DeprecatedElectionProof
+ PoStRandomness
+ PoStPartialTickets
+ PoStSectorIDs
+ PoStChallengeIDXs
+ PoStProof

### 构造区块的函数

```
// Generate returns a new block created from the messages in the pool.
func (w *DefaultWorker) Generate(
	ctx context.Context,
	baseTipSet block.TipSet,
	ticket block.Ticket,
	nullBlockCount uint64,
	postRandomness []byte,
	winners []*proofs.EPoStCandidate,
	post []byte,
) (*block.Block, error) 
```

基础字段

|      字段       |                                  赋值                                   |
| :-------------: | :---------------------------------------------------------------------: |
|     Miner:      |                              w.minerAddr,                               |
|     Height:     |                       types.Uint64(blockHeight),                        |
|    Parents:     |                            baseTipSet.Key(),                            |
|  ParentWeight   |               weight, err := w.getWeight(ctx, baseTipSet)               |
|     Height      |             blockHeight := baseHeight + nullBlockCount + 1              |
|    Timestamp    |                              w.clock.Now()                              |
|    BlockSig     |       w.workerSigner.SignBytes(next.SignatureData(), workerAddr)        |
| BLSAggregateSig | unwrappedBLSMessages, blsAggregateSig, err := aggregateBLS(blsAccepted) |
|    cachedCid    |                                                                         |
|   cachedBytes   |                                                                         |

业务信息

```go
pending := w.messageSource.Pending()
mq := NewMessageQueue(pending)
candidateMsgs := orderMessageCandidates(mq.Drain())
results, err := w.processor.ApplyMessagesAndPayRewards(ctx, stateTree, vms, types.UnwrapSigned(candidateMsgs),
   w.minerOwnerAddr, types.NewBlockHeight(blockHeight), ancestors)
   for i, r := range results {
   msg := candidateMsgs[i]
   if r.Failure == nil {
      if msg.Message.From.Protocol() == address.BLS {
         blsAccepted = append(blsAccepted, msg)
      } else {
         secpAccepted = append(secpAccepted, msg)
      }
   } else if r.FailureIsPermanent {
      log.Infof("permanent ApplyMessage failure, [%s] (%s)", msg, r.Failure)
      mc, err := msg.Cid()
      if err == nil {
         w.messageSource.Remove(mc)
      } else {
         log.Warnf("failed to get CID from message", err)
      }
   } else {
      log.Infof("temporary ApplyMessage failure, [%s] (%s)", msg, r.Failure)
   }
}
unwrappedBLSMessages, blsAggregateSig, err := aggregateBLS(blsAccepted)
txMeta, err := w.messageStore.StoreMessages(ctx, secpAccepted, unwrappedBLSMessages)
   baseReceiptRoot, err := w.tsMetadata.GetTipSetReceiptsRoot(baseTipSet.Key())
   baseStateRoot, err := w.tsMetadata.GetTipSetStateRoot(baseTipSet.Key())
```

|      字段       |      赋值       |
| :-------------: | :-------------: |
|    Messages     |     txMeta      |
| MessageReceipts | baseReceiptRoot |
|    StateRoot    |  baseStateRoot  |

挖矿信息:

```go
winningSectorIDs := make([]types.Uint64, len(winners))
winningChallengeIndexes := make([]types.Uint64, len(winners))
winningPartialTickets := make([][]byte, len(winners))
for i, candidate := range winners {
   winningSectorIDs[i] = types.Uint64(candidate.SectorID)
   winningChallengeIndexes[i] = types.Uint64(candidate.SectorChallengeIndex)
   winningPartialTickets[i] = candidate.PartialTicket
}
```

|          字段           |          赋值           |
| :---------------------: | :---------------------: |
|         Ticket:         |         ticket          |
| DeprecatedElectionProof |                         |
|     PoStRandomness      |     postRandomness      |
|   PoStPartialTickets    |  winningPartialTickets  |
|      PoStSectorIDs      |    winningSectorIDs     |
|    PoStChallengeIDXs    | winningChallengeIndexes |
|        PoStProof        |          post           |

## 区块生产以后的处理

StartMining调用handleNewMiningOutput来处理outCh管道中的区块,handleNewMiningOutput最后调用AddNewBlock来处理区块的相关信息

```go
// AddNewBlock接收一个新挖掘的块，并将其存储，验证并传播到网络。
func (node *Node) AddNewBlock(ctx context.Context, b *block.Block) (err error) {
	ctx, span := trace.StartSpan(ctx, "Node.AddNewBlock")
	span.AddAttributes(trace.StringAttribute("block", b.Cid().String()))
	defer tracing.AddErrorEndSpan(ctx, span, &err)

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

|      功能      |                              代码                               |
| :------------: | :-------------------------------------------------------------: |
|      存储      |      blkCid, err := node.Blockstore.CborStore.Put(ctx, b)       |
|      传播      | err = node.syncer.BlockTopic.Publish(ctx, b.ToNode().RawData()) |
| 链处理区块信息 |  node.syncer.ChainSyncManager.BlockProposer().SendOwnBlock(ci)  |
