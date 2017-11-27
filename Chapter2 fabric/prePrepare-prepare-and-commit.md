## 待完善。。。。。。
## 事件来源
在serve()函数中提到的`NewPeerWithEngine()` >> `engFactory(peer)<=>GetEngine(peer)`中，该函数位于`engine.go`中，`for`循环不断通过`engine.consensusFan.GetOutChannel()`从`messageFan`中获取`msg`，然后通过`engine.consenter.RecvMsg(msg.Msg, msg.Sender)`发送出去,该函数实际上由`externalEventReceiver`实现。
```go
// RecvMsg is called by the stack when a new message is received
func (eer *externalEventReceiver) RecvMsg(ocMsg *pb.Message, senderHandle *pb.PeerID) error {
	eer.manager.Queue() <- batchMessageEvent{
		msg:    ocMsg,
		sender: senderHandle,
	}
	return nil
}
```
注意到，该函数是将batchMessageEvent发送到err.manager.Queue()中。

## 事件处理
* 在`obcBatch`初始化时通过`goroutine`调用`managerImpl.eventLoop()`，最终会调用`obcBatch.ProcessEvent()`。
* 在`obcBatch.ProcessEvent()`函数中，通过`switch et := event.(type)` 分支处理`events`：
### 1、batchMessageEvent
```go
case batchMessageEvent:
		ocMsg := et
		return op.processMessage(ocMsg.msg, ocMsg.sender)
```
注意到，该分支会调用`obcBatch.processMessage(batchMessageEvent.msg, batchMessageEvent.sender)`
```go
func (op *obcBatch) processMessage(ocMsg *pb.Message, senderHandle *pb.PeerID) events.Event {
	if ocMsg.Type == pb.Message_CHAIN_TRANSACTION {
		req := op.txToReq(ocMsg.Payload)
		return op.submitToLeader(req)
	}

	if ocMsg.Type != pb.Message_CONSENSUS {
		logger.Errorf("Unexpected message type: %s", ocMsg.Type)
		return nil
	}

	batchMsg := &BatchMessage{}
	...

	if req := batchMsg.GetRequest(); req != nil {
		...
		if (op.pbft.primary(op.pbft.view) == op.pbft.id) && op.pbft.activeView {
			return op.leaderProcReq(req)
		}
		op.startTimerIfOutstandingRequests()
		return nil
	} else if pbftMsg := batchMsg.GetPbftMessage(); pbftMsg != nil {
		senderID, err := getValidatorID(senderHandle) // who sent this?
		if err != nil {
			panic("Cannot map sender's PeerID to a valid replica ID")
		}
		msg := &Message{}
		err = proto.Unmarshal(pbftMsg, msg)
		if err != nil {
			logger.Errorf("Error unpacking payload from message: %s", err)
			return nil
		}
		return pbftMessageEvent{
			msg:    msg,
			sender: senderID,
		}
	}

	logger.Errorf("Unknown request: %+v", batchMsg)

	return nil
}
```
注意到：
* 1.1 消息是`Message_CHAIN_TRANSACTION`，将消息的`payload`封装成`Request`，交由`submitToLeader(req)`处理，经过广播等操作，
`submitToLeader(req)` >> `leaderProcReq(req)` >> `sendBatch()` 最终返回`events.Event`类型的`RequestBatch`，这个事件在`pbft.ProcessEvent(event)`中处理。
* 1.2 消息是`Message_CONSENSUS`，进行分支：
    * 1.2.1 `req`，最终会调用`leaderProcReq(req)`，后期同`Message_CHAIN_TRANSACTION`
    * 1.2.2 `pbftMsg`，经过处理，最终返回`pbftMessageEvent`，这个事件在`pbft.ProcessEvent(event)`中处理。

### 2、default，会执行`pbft.ProcessEvent(event)`
在该函数中通过`switch et := e.(type)`进行分支操作：

#### 2.1 pbftMessageEvent
它会执行`next, err := instance.recvMsg(msg.msg, msg.sender)`，该函数如下：
```go
func (instance *pbftCore) recvMsg(msg *Message, senderID uint64) (interface{}, error) {
	if reqBatch := msg.GetRequestBatch(); reqBatch != nil {
		return reqBatch, nil
	} else if preprep := msg.GetPrePrepare(); preprep != nil {
		if senderID != preprep.ReplicaId {
			return nil, fmt.Errorf("Sender ID included in pre-prepare message (%v) doesn't match ID corresponding to the receiving stream (%v)", preprep.ReplicaId, senderID)
		}
		return preprep, nil
	} else if prep := msg.GetPrepare(); prep != nil {
		if senderID != prep.ReplicaId {
			return nil, fmt.Errorf("Sender ID included in prepare message (%v) doesn't match ID corresponding to the receiving stream (%v)", prep.ReplicaId, senderID)
		}
		return prep, nil
	} else if commit := msg.GetCommit(); commit != nil {
		if senderID != commit.ReplicaId {
			return nil, fmt.Errorf("Sender ID included in commit message (%v) doesn't match ID corresponding to the receiving stream (%v)", commit.ReplicaId, senderID)
		}
		return commit, nil
	} else if chkpt := msg.GetCheckpoint(); chkpt != nil {
		if senderID != chkpt.ReplicaId {
			return nil, fmt.Errorf("Sender ID included in checkpoint message (%v) doesn't match ID corresponding to the receiving stream (%v)", chkpt.ReplicaId, senderID)
		}
		return chkpt, nil
	} else if vc := msg.GetViewChange(); vc != nil {
		if senderID != vc.ReplicaId {
			return nil, fmt.Errorf("Sender ID included in view-change message (%v) doesn't match ID corresponding to the receiving stream (%v)", vc.ReplicaId, senderID)
		}
		return vc, nil
	} else if nv := msg.GetNewView(); nv != nil {
		if senderID != nv.ReplicaId {
			return nil, fmt.Errorf("Sender ID included in new-view message (%v) doesn't match ID corresponding to the receiving stream (%v)", nv.ReplicaId, senderID)
		}
		return nv, nil
	} else if fr := msg.GetFetchRequestBatch(); fr != nil {
		if senderID != fr.ReplicaId {
			return nil, fmt.Errorf("Sender ID included in fetch-request-batch message (%v) doesn't match ID corresponding to the receiving stream (%v)", fr.ReplicaId, senderID)
		}
		return fr, nil
	} else if reqBatch := msg.GetReturnRequestBatch(); reqBatch != nil {
		// it's ok for sender ID and replica ID to differ; we're sending the original request message
		return returnRequestBatchEvent(reqBatch), nil
	}
	return nil, fmt.Errorf("Invalid message: %v", msg)
}
```
从代码可以看出，该函数返回各种类型的时间，供`pbft.ProcessEvent(event)`处理。

#### 2.2、RequestBatch
它会执行`recvRequestBatch(reqBatch)`，进一步调用`sendPrePrepare(reqBatch, digest)`
```go
func (instance *pbftCore) sendPrePrepare(reqBatch *RequestBatch, digest string) {
	...
	logger.Debugf("Primary %d broadcasting pre-prepare for view=%d/seqNo=%d and digest %s", instance.id, instance.view, n, digest)
	instance.seqNo = n
	preprep := &PrePrepare{
		View:           instance.view,
		SequenceNumber: n,
		BatchDigest:    digest,
		RequestBatch:   reqBatch,
		ReplicaId:      instance.id,
	}
	cert := instance.getCert(instance.view, n)
	cert.prePrepare = preprep
	cert.digest = digest
	instance.persistQSet()
	instance.innerBroadcast(&Message{Payload: &Message_PrePrepare{PrePrepare: preprep}})
	instance.maybeSendCommit(digest, instance.view, n)
}
```
注意到：
* 1） 初始化`PrePrepare`，赋值给`cert`的成员变量`prePrepare`(指针赋值)，同时作为`innerBroadcast()`的参数，广播出去。
* 2）`innerBroadcast()`会随机模拟`byzantine fault`节点，正常节点会进一步调用`instance.consumer.broadcast(msgRaw)`
(该函数由`obcBatch`实现)，最终调用`broadcaster.send()`实现广播。
* 3）调用`instance.maybeSendCommit(digest, instance.view, n)`，因为if条件不成立，没有进行实质性操作。

#### 2.3、PrePrepare
它会执行`recvPrePrepare(prePrepare)`
```go
func (instance *pbftCore) recvPrePrepare(preprep *PrePrepare) error {
    ...
    if ... {
		logger.Debugf("Backup %d broadcasting prepare for view=%d/seqNo=%d", instance.id, preprep.View, preprep.SequenceNumber)
		prep := &Prepare{
			View:           preprep.View,
			SequenceNumber: preprep.SequenceNumber,
			BatchDigest:    preprep.BatchDigest,
			ReplicaId:      instance.id,
		}
		cert.sentPrepare = true
		instance.persistQSet()
		instance.recvPrepare(prep)
		return instance.innerBroadcast(&Message{Payload: &Message_Prepare{Prepare: prep}})
	}

	return nil
}
```
从函数可看出：
* 1）初始化`Prepare`，并作为`recvPrepare()`和`innerBroadcast()`的参数。
* 2）`recvPrepare()`对`prepare`进行验证，调用`instance.maybeSendCommit(prep.BatchDigest, prep.View, prep.SequenceNumber)`，在`Prepare`中介绍。
* 3）`innerBroadcast()`同2.1（2）

#### 2.4、Prepare
它会执行`instance.recvPrepare(prepare)`,调用`instance.maybeSendCommit(prep.BatchDigest, prep.View, prep.SequenceNumber)`
```go
func (instance *pbftCore) maybeSendCommit(digest string, v uint64, n uint64) error {
	cert := instance.getCert(v, n)
	if instance.prepared(digest, v, n) && !cert.sentCommit {
		logger.Debugf("Replica %d broadcasting commit for view=%d/seqNo=%d",
			instance.id, v, n)
		commit := &Commit{
			View:           v,
			SequenceNumber: n,
			BatchDigest:    digest,
			ReplicaId:      instance.id,
		}
		cert.sentCommit = true
		instance.recvCommit(commit)
		return instance.innerBroadcast(&Message{&Message_Commit{commit}})
	}
	return nil
}
```
注意到，在`RequestBatch`虽然有被调用，但是因为条件不成立，所以无法进入`if`分支。在接收到`prePrepare`和`prepare`后，条件成立，进入`if`分支。
* 1）初始化`commit`，且仅同一个`prepare`只初始化一次。
* 2）调用`instance.recvCommit(commit)`，在`Commit`中讲解。
* 3）`innerBroadcast()`同2.1（2）

#### 2.5、Commit
它会执行`instance.recvCommit(commit)`，进一步调用`instance.committed(commit.BatchDigest, commit.View, commit.SequenceNumber)`
```go
func (instance *pbftCore) committed(digest string, v uint64, n uint64) bool {
	if !instance.prepared(digest, v, n) {
		return false
	}

	quorum := 0
	cert := instance.certStore[msgID{v, n}]
	if cert == nil {
		return false
	}

	for _, p := range cert.commit {
		if p.View == v && p.SequenceNumber == n {
			quorum++
		}
	}

	logger.Debugf("Replica %d commit count for view=%d/seqNo=%d: %d",
		instance.id, v, n, quorum)

	return quorum >= instance.intersectionQuorum()
}

func (instance *pbftCore) intersectionQuorum() int {
	return (instance.N + instance.f + 2) / 2
}
```
从代码可以看出，通过`cert`中保存的`commit`判定最终是否`committed`。`cert`的结构如下：
```go
ype msgCert struct {
	digest      string
	prePrepare  *PrePrepare
	sentPrepare bool
	prepare     []*Prepare
	sentCommit  bool
	commit      []*Commit
}
```

## committed的后续操作尚未研究

