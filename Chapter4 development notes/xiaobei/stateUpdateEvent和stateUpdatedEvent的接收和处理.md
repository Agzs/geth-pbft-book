# 相关结构体、事件的定义
在pbtf-core.go里加入结构体`stateUpdateTarget`。在`func (instance *pbftCore) recvCheckpoint(chkpt *types.Checkpoint) events.Event`里，当在`checkpointStore`里存储的checkpoint的`SequenceNumber`、`ID`与接收到的checkpoint的`SequenceNumber`、`ID`相同的checkpoint的个数等于f+1的时候，我们对这个checkpoint就有了一个弱认证，这时候就会调用`func (instance *pbftCore) witnessCheckpointWeakCert(chkpt *types.Checkpoint)`,在这个函数里初始化了一个`stateUpdateTarget`：
```
	target := &stateUpdateTarget{
		checkpointMessage: checkpointMessage{
			seqNo: chkpt.SequenceNumber,
			id:    snapshotID,
		},
		replicas: checkpointMembers,
	}
	instance.updateHighStateTarget(target)
```
其中，`replicas`存储了对checkpoint达成共识的f+1个节点的ID，`checkpointMessage`存储了达成共识的checkpoint。当`instance.skipInProgress`为true的时候进行状态转换，把`target`传入`func (instance *pbftCore) retryStateTransfer(optional *stateUpdateTarget)`,最终在该函数里经过`instance.helper.skipTo(target.seqNo, target.id, target.replicas)`调用`func (helper *Helper) UpdateState(tag interface{}, target *types.BlockchainInfo, peers []*types.PeerID)`进行状态更新，把`stateUpdateEvent`放入了`Queue()`里。
# stateUpdateEvent的处理过程
在fabric里面，对`stateUpdateEvent`的处理是在executor.go的`func (co *coordinatorImpl) ProcessEvent(event events.Event) events.Event`里。在这个函数里面通过调用peer/statetransfer.go下的`func (sts *coordinatorImpl) SyncToTarget(blockNumber uint64, blockHash []byte, peerIDs []*pb.PeerID) (error, bool)`实现`peerIDs`里面指定的节点的区块同步（这里指的是那些获得弱认证的节点）。由于在geth-pbft里面我们沿用了以太坊里的区块同步功能，所以在geth-pbft里的pbft-core.go里把区块同步的功能给注释掉了。

在geth-pbft里把对`stateUpdateEvent`的处理放在了pbft-core.go里，去掉了原来的区块同步的实现，通过调用`instance.helper.StateUpdated(et.tag, info)`把`checkpointmessage`传进去进而进行checkpoint的状态更新。
# stateUpdatedEvent的接收和处理
在helper.go下的`func (helper *Helper) StateUpdated(tag interface{}, target *types.BlockchainInfo)`里把`stateUpdatedEvent`放入了`manager.Queue()`，经过层层调用后该事件同样是在`func (instance *pbftCore) ProcessEvent(e events.Event) events.Event`里被处理。

在`stateUpdatedEvent`里传递过来的`checkpointMessage.seqNo`如果小于本节点的低水位，那么就判断`instance.highStateTarget`是否为空。
`instance.highStateTarget`里存放了当前结点达到弱认证的checkpoint的信息以及使该节点得到checkpoint弱认证的其他节点的ID。如果`instance.highStateTarget`不为空，但是触发`stateUpdatedEvent`的`checkpointMessage.seqNo`小于`instance.highStateTarget.seqNo`，那么将会重新触发'instance.retryStateTransfer(nil)'进行状态转换。只要是在`stateUpdatedEvent`里传递过来的`checkpointMessage.seqNo`如果小于本节点的低水位，那么就返回nil，否则就执行以后的操作——给`instance.lastExec`赋值为触发`stateUpdatedEvent`的`checkpointMessage.seqNo`、在`func (instance *pbftCore) moveWatermarks(n uint64)`里删除各种store存储的小于低水位的block的信息等。


