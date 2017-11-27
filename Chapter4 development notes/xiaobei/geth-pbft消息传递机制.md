# geth-pbft消息传递入口
1、在`func CreateConsensusEngine(ctx *node.ServiceContext, config *Config, chainConfig *params.ChainConfig, db ethdb.Database, pm *ProtocolManager) consensus.Engine `里选择pbft共识机制：
```
	if chainConfig.PBFT != nil {
		pbftEngine := pbft.New(config.PbftId, db)
		pm.commChan = pbftEngine.commChan
		return pbftEngine
	}
```
2、将`engine`初始化为PBFT后，在`miner/agent.go`的`func (self *CpuAgent) mine(work *Work, stop <-chan struct{})`里调用`PBFT`下的`Seal`进行挖矿：
```
func (self *CpuAgent) mine(work *Work, stop <-chan struct{}) {
	if result, err := self.engine.Seal(self.chain, work.Block, stop); result != nil {
		log.Info("Successfully sealed new block", "number", result.Number(), "hash", result.Hash())
		self.returnCh <- &Result{work, result}
	} else {
		if err != nil {
			log.Warn("Block sealing failed", "err", err)
		}
		self.returnCh <- nil
	}
}
```
3、在`func (c *PBFT) Seal(chain consensus.ChainReader, block *types.Block, stop <-chan struct{}) (*types.Block, error)`里挖到块后，将挖到的块传递给`func (instance *pbftCore) sendPrePrepare(block *types.Block)`创建PrePrepare信息，这就是geth-pbft共识机制消息传递的入口。
# geth-pbft共识消息传递过程
1、`func (instance *pbftCore) sendPrePrepare(block *types.Block)`接收到一个`block`之后初始化了`preprep`信息，然后在`certStore`里循环遍历所有的`prePrepare`消息看是否存在拥有相同`view`、相同`digest`但是拥有序号不同的区块，如果有的话就`return`；然后再经过其他一系列的判断（如判断n是否在高低水位之间等）后如果无问题就给该`block`颁发一个`prePrepare`证书，并把`prePrepare`消息放入到`commChan`。

2、在`func (instance *pbftCore) ProcessEvent(e events.Event) events.Event`里对传递过来的共识消息的类型进行判断并执行不同的操作。下面以收到`prePrepare`消息为例：
```
	case *types.PrePrepare:
		err = instance.recvPrePrepare(et)
```
会调用`func (instance *pbftCore) recvPrePrepare(preprep *types.PrePrepare) error `，在该函数里面会对`PrePrepare`消息进行验证（如判断该验证是否发生的了_viewchange_、该`prePrepare`的信息对应的主节点是否发生了改变、`certStore`里是否拥有与收到的`prePrepare`消息拥有相同`view/seqNo`但不同`digest`的信息等）。如果验证无误后，就把收到的`prePrepare`消息放入到`certStore`里。接着在`if`语句里以非主节点的身份发送处于`prePrepared`状态的还未发送`Prepare`的消息：
```
if instance.primary(instance.view) != instance.id && instance.prePrepared(string(preprep.BlockHash), preprep.View, preprep.SequenceNumber) && !cert.sentPrepare {
		logger.Debugf("Backup %d broadcasting prepare for view=%d/seqNo=%d", instance.id, preprep.View, preprep.SequenceNumber)
		prep := &types.Prepare{
			View:           preprep.View,
			SequenceNumber: preprep.SequenceNumber,
			BlockHash:      preprep.BlockHash,
			ReplicaId:      instance.id,
		}
		cert.sentPrepare = true
		instance.recvPrepare(prep)
		instance.commChan <- &types.PbftMessage{Payload: &types.PbftMessage_Prepare{Prepare: prep}} /// use channel to send prepare to ProtocolManager. Zhiguo
		return nil
	}
```
将`prepare`消息放入commchan等待广播。

3、注意到在`func (instance *pbftCore) recvPrePrepare(preprep *types.PrePrepare) error `里调用了`func (instance *pbftCore) recvPrepare(prep *types.Prepare) error`，在这个函数里首先对该`prepare`消息进行判断，看该消息是否来自于主节点。然后取出该`prepare`所认证的`block`对应的证书`cert`：
```
	cert := instance.getCert(prep.View, prep.SequenceNumber)

	for _, prevPrep := range cert.prepare {
		if prevPrep.ReplicaId == prep.ReplicaId {
			logger.Warningf("Ignoring duplicate prepare from %d", prep.ReplicaId)
			return nil
		}
	}
	cert.prepare = append(cert.prepare, prep)
        return instance.maybeSendCommit(string(prep.BlockHash), prep.View, prep.SequenceNumber)
```
> 注意`cert := instance.getCert(prep.View, prep.SequenceNumber)`返回的证书`cert`的数据结构是: 
> ```
> type msgCert struct {
>  	digest      string
>  	prePrepare  *types.PrePrepare
> 	sentPrepare bool
>  	prepare     []*types.Prepare
>  	sentCommit  bool
>  	commit      [] *types.Commit
> }
> ```
> 其中`prepare`和`commit`都是指针数组，因为里面存放的不止是该节点自己发起的`prepare`和`commit`消息，还有收到的来自其他节点的`prepare`和`commit`消息，对这些消息的数量进行判断来判断该`block`是否处于`prepared`或者是`commited`的状态。
在`for`循环里判断在`cert.prepare[]`是否已存在来自本节点的`prepare`消息，如果不存在就将该`prepare`消息存储到`cert.prepare`数组里，然后接着调用`func (instance *pbftCore) maybeSendCommit(digest string, v uint64, n uint64) error`。

4、在`func (instance *pbftCore) maybeSendCommit(digest string, v uint64, n uint64) error`里首先获得block对应的证书,然后判断该block是否已经处于prepared状态并且还未发送commit信息：
```
func (instance *pbftCore) maybeSendCommit(digest string, v uint64, n uint64) error {
	cert := instance.getCert(v, n)
	if instance.prepared(digest, v, n) && !cert.sentCommit {
		logger.Debugf("Replica %d broadcasting commit for view=%d/seqNo=%d",
			instance.id, v, n)
		commit := &types.Commit{
			View:           v,
			SequenceNumber: n,
			BlockHash:      unsafe.StringBytes(digest),
			ReplicaId:      instance.id,
		}
		cert.sentCommit = true
		instance.recvCommit(commit)
		instance.commChan <- &types.PbftMessage{Payload: &types.PbftMessage_Commit{Commit: commit}} /// use channel to send commit to ProtocolManager. Zhiguo
		///return instance.innerBroadcast(&Message{&Message_Commit{commit}})
	}
	return nil
}
```
> 在这个函数里是调用`instance.prepared(digest, v, n)`来判断block是否处于prepaed状态的，判断标准是:
```
func (instance *pbftCore) prepared(digest string, v uint64, n uint64) bool {
	if !instance.prePrepared(digest, v, n) {
		return false
	}

	if p, ok := instance.pset[n]; ok && p.View == v && p.BlockHash == digest {
		return true
	}

	quorum := 0
	cert := instance.certStore[msgID{v, n}]
	if cert == nil {
		return false
	}

	for _, p := range cert.prepare {
		if p.View == v && p.SequenceNumber == n && string(p.BlockHash[:]) == digest {
			quorum++
		}
	}

	logger.Debugf("Replica %d prepare count for view=%d/seqNo=%d: %d",
		instance.id, v, n, quorum)

	return quorum >= instance.intersectionQuorum()-1
}
```
> 判断方法有两个：
> 一是看给block分配的序号n是否在pset里面存在，pset里面存着节点在上一 view 中达到prepared状态的请求的一些信息，如果存在并且view和blockhash都没有问题则说明该block已经处于prepared状态，返回ture。
> 二是判断该节点是否收到一定数量的prepare消息，如果满足一定的数量要求则返回ture。数量要求在以下函数里定义。
>```
> func (instance *pbftCore) intersectionQuorum() int {
> 	return (instance.N + instance.f + 2) / 2
> }
>```
当判断block已经处于prepared状态后开始初始化commit，然后将commit消息放入`commChan`里面。

5、将commit消息放入`commChan`里面之前会调用`func (instance *pbftCore) recvCommit(commit *types.Commit) error`，类似于`func recvPrepare()`和`func recvPrePrepare()`,在该函数里将commit消息放入了`cert.commit`数组里`cert.commit = append(cert.commit, commit)`,然后在`commit`数组里判断是否收到了一定数量的commit消息，如果有则整个共识消息的传递过程结束，将一个结构体放入`finishChan`来通知PBFT共识已经完成。
```
	if instance.committed(string(commit.BlockHash), commit.View, commit.SequenceNumber) {
		instance.stopTimer()
		instance.lastNewViewTimeout = instance.newViewTimeout
		delete(instance.outstandingBlock, string(commit.BlockHash))

		///		instance.executeOutstanding()
		instance.finishedChan <- struct{}{} /// inform PBFT consensus is reached.  --Zhiguo

		if commit.SequenceNumber == instance.viewChangeSeqNo {
			logger.Infof("Replica %d cycling view for seqNo=%d", instance.id, commit.SequenceNumber)
			instance.sendViewChange()
		}
	}

```