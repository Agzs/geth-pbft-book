# 入口
在fabric里面，当接收到一个`committedEvent`的时候，将会返回一个`execDoneEvent{}`,然后再把该event交给`pbft.ProcessEvent(event)`进行处理，然后再调用`func (instance *pbftCore) execDoneSync()`,再这里面调用了`instance.Checkpoint(instance.lastExec, instance.consumer.getState())`。这就是`func (instance *pbftCore) Checkpoint(seqNo uint64, id []byte)`的入口。

在geth-pbft的`func (instance *pbftCore) recvCommit(commit *types.Commit) error`里面，我在当该block处于`committed`状态的时候，增加语句`instance.helper.manager.Queue() <- execDoneEvent{}`,然后在`pbft.ProcessEvent(event)`里面增加对这个事件的处理：
```
case execDoneEvent:
		instance.execDoneSync()
		if instance.skipInProgress {
			instance.retryStateTransfer(nil)
		}
		// We will delay new view processing sometimes
		return instance.processNewView()
```
恢复`func (instance *pbftCore) execDoneSync()`，在该函数里，当当前处理的block的序号是instance.K的整数倍的时候，调用`instance.Checkpoint(instance.lastExec, instance.consumer.getState())`，初始化checkpoint信息。
# checkpoint部分的恢复与修改
1、 恢复`func (instance *pbftCore) Checkpoint(seqNo uint64, id []byte)`,在这个函数里初始化了`checkpoint`信息，把它放入了字典`instance.chkpts`（key是`seqNo`，value值是string类型的，是checkpoint的digest），并把`checkpoint`信息进行了广播；调用了`instance.recvCheckpoint(chkpt)`。
> `checkpointStore`里存储的是本节点的和来自其他节点的checkpoint，而`chkpts`存储的只有本节点的关于checkpoint的信息。

2、恢复`func (instance *pbftCore) recvCheckpoint(chkpt *types.Checkpoint) events.Event`，在这个函数里对收到的checkpoint信息进行验证后把它放入了`checkpointStore`里，并对`checkpointStore`里等于`chkpt.SequenceNumber`、`chkpt.Id`的checkpoint个数进行统计并做出相应处理。如果checkpoint的个数大于2f+1个，将会触发`func (instance *pbftCore) processNewView() events.Event`。根据`func (instance *pbftCore) recvCheckpoint(chkpt *types.Checkpoint) events.Event`的内部调用：
* 恢复了`func (instance *pbftCore) weakCheckpointSetOutOfRange(chkpt *types.Checkpoint) bool`
* 恢复了`func (instance *pbftCore) witnessCheckpointWeakCert(chkpt *Checkpoint)`

2.1、恢复了`func (instance *pbftCore) weakCheckpointSetOutOfRange(chkpt *types.Checkpoint) bool`，在这个函数进行判断：
* 如果`checkpoint.seqNo`小于本节点的高水位把它从`instance.hChkpts`中删除，并`return false`。该字典存储那些高于本节点的高水位的`seqNo`,key是`ReplicaId`,value是`seqNo`。否则`return true`。
* 如果`checkpoint.seqNo`大于本届点的高水位，并且`instance.hChkpts`中存储的高于本节点高水位的`checkpoint.seqNo`的个数小于f+1个，则把他放入`instance.hChkpts`并`return false`。如果`instance.hChkpts`中存储的高于本节点高水位的`checkpoint.seqNo`的个数大于f+1个，意味着在同一时刻该节点是收集不够`2f+1`条针对该`seqNo`的`checkpoint`消息的，那么该checkpoint将永远达不到stable的状态，因为3f+1-（f+1）=2f<(2f+1)。在这种情况下，我们将从`instance.blockStore`里清空所有已处理的block，从`instance.outstandingBlocks`里清空所有等待处理的block，重新设置高低水位，重新设置checkpoint，并`return true`。在状态重置的过程中恢复了`func (instance *pbftCore) invalidateState`、`func (instance *pbftCore) moveWatermarks(n uint64)`等函数。

2.1.1、 在`func (instance *pbftCore) weakCheckpointSetOutOfRange(chkpt *types.Checkpoint) bool`调用`func (instance *pbftCore) moveWatermarks(n uint64)`，恢复该函数。该函数重新设置了低水位，清空了`certStore`、`checkpointStore`、`pset`、`qset`、`chkpts`里seqNo低于低水位的block的信息。

2.1.2、 在`func (instance *pbftCore) moveWatermarks(n uint64)`里调用`func (instance *pbftCore) resubmitRequestBatches()`,恢复更改为`func (instance *pbftCore) resubmitBlockMsges()`。具体更改内容参见该函数。

2.1.3、在`func (instance *pbftCore) resubmitBlockMsges()`调用了`instance.recvBlockMsg(block)`，恢复`func (instance *pbftCore) recvBlockMsg(block *types.Block) error`。在该函数里调用`instance.sendPrePrepare(block)`把block以`PrePrepareMessage`的形式广播出去。

2.2、恢复了`func (instance *pbftCore) witnessCheckpointWeakCert(chkpt *Checkpoint)`，当在已存储`checkpointstore`里`sqeNo`和`id`都和接收到的checkpoint的`sqeNo`和`id`相同的checkpoint的个数等于f+1时，就说明该checkpoint在该节点上得到了一个weak certificates，就会调用这个函数。

2.2.1、在`func (instance *pbftCore) witnessCheckpointWeakCert(chkpt *Checkpoint)`里，初始化了一个`target`,他是`stateUpdateTarget`类型的，通过在`func (instance *pbftCore) updateHighStateTarget(target *stateUpdateTarget)`把`target`传递给`instance.highStateTarget = target`,获得弱认证的checkpoint的证书。因此恢复`func (instance *pbftCore) updateHighStateTarget(target *stateUpdateTarget) `。

2.2.2、在`func (instance *pbftCore) witnessCheckpointWeakCert(chkpt *types.Checkpoint)`里，当`instance.skipInProgress`为true的时候进行状态转换，恢复`func (instance *pbftCore) retryStateTransfer(optional *stateUpdateTarget)`。

2.2.3、在`func (instance *pbftCore) retryStateTransfer(optional *stateUpdateTarget)`里经过一系列的验证后，调用`instance.consumer.skipTo(target.seqNo, target.id, target.replicas)`,这里把fabric里面的`func (op *obcGeneric) skipTo(seqNo uint64, id []byte, replicas []uint64)`拿到了pbft-core.go里，变为`func (instance *pbftCore) skipTo(seqNo uint64, id []byte, replicas []uint64)`,并且在pbft_message.go里定义了`type BlockchainInfo struct`,实现了`message`接口。

2.2.4、在`func (instance *pbftCore) skipTo(seqNo uint64, id []byte, replicas []uint64)`里又进一步调用了`instance.helper.UpdateState(&checkpointMessage{seqNo, id}, info, getValidatorHandles(replicas))`,把`checkpointMessage`用`stateUpdateEvent`封装放入了manager.Queue()里。这里把这个函数恢复到了helper.go里，其中参数`getValidatorHandles(replicas)`也进行了恢复。

2.2.5、在函数这里把`BlockchainInfo`、`PeerID`都恢复到了pbft_message.go里。

