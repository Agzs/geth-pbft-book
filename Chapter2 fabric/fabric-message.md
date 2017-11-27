我们要实现的`peer.go`中函数接受`*pb.Message`类型的消息，消息定义在`fabric.pb.go`中   
```
type Message struct {  
        Type      Message_Type  
        Timestamp *google_protobuf.Timestamp  
        Payload   []byte  
        Signature []byte    
}
```
其中`Type`为消息类型，其值为`uint`类型，`Timestamp`为时间戳，`Payload`为消息内容的编码，`Signature`为签名。  
Payload消息内容编码之前有一下几种类型：
```

type Message_RequestBatch struct {
	RequestBatch *RequestBatch
}
type Message_PrePrepare struct {
	PrePrepare *PrePrepare 
}
type Message_Prepare struct {
	Prepare *Prepare 
}
type Message_Commit struct {
	Commit *Commit 
}
type Message_Checkpoint struct {
	Checkpoint *Checkpoint 
}
type Message_ViewChange struct {
	ViewChange *ViewChange 
}
type Message_NewView struct {
	NewView *NewView 
}
type Message_FetchRequestBatch struct {
	FetchRequestBatch *FetchRequestBatch
}
type Message_ReturnRequestBatch struct {
	ReturnRequestBatch *RequestBatch 
}

```
1、RequestBatch类型的
`batch.go` 145行：给出Payload和Type
```

func (op *obcBatch) broadcastMsg(msg *BatchMessage) {
	msgPayload, _ := proto.Marshal(msg)
	ocMsg := &pb.Message{
		Type:    pb.Message_CONSENSUS,
		Payload: msgPayload,
	}
	op.broadcaster.Broadcast(ocMsg)
}

```
`batch.go` 135行：Payload取自这儿
```
func (op *obcBatch) submitToLeader(req *Request) events.Event {
	...
	op.broadcastMsg(&BatchMessage{Payload: &BatchMessage_Request{Request: req}})
	...
}

```
`batch.go` 445行：打包成批
```
func (op *obcBatch) wrapMessage(msgPayload []byte) *pb.Message {
	batchMsg := &BatchMessage{Payload: &BatchMessage_PbftMessage{PbftMessage: msgPayload}}
	packedBatchMsg, _ := proto.Marshal(batchMsg)
	ocMsg := &pb.Message{
		Type:    pb.Message_CONSENSUS,
		Payload: packedBatchMsg,
	}
	return ocMsg
}
```
2、Preprepare类型的
pbft-core 659行：此处获得Payload
```

func (instance *pbftCore) sendPrePrepare(reqBatch *RequestBatch, digest string) {
        ...
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
	instance.innerBroadcast(&Message{Payload: &Message_PrePrepare{PrePrepare: preprep}})
	...
}

```
`batch.go` 445行：对Payload进行进一步的包装并给出消息类型Type
```
func (op *obcBatch) wrapMessage(msgPayload []byte) *pb.Message {
	batchMsg := &BatchMessage{Payload: &BatchMessage_PbftMessage{PbftMessage: msgPayload}}
	packedBatchMsg, _ := proto.Marshal(batchMsg)
	ocMsg := &pb.Message{
		Type:    pb.Message_CONSENSUS,
		Payload: packedBatchMsg,
	}
	return ocMsg
}
```
3、`Prepare`、`commit`、`Checkpoint`、`ViewChange`类型的和`Preprepare`消息产生的过程类似，   
`pbft-core.go` 693行  `recvPrePrepare`方法中获得`Prepare`类型的`Payload`，  
`pbft-core.go` 798行  `maybeSendCommit`方法中获得`commit`类型的`Payload`，    
 `pbft-core.go` 958行  `Checkpoint`方法中获得`checkpoint`类型的`Payload`，  
`viewchange.go` 125行  `sendViewChange`方法中获得`ViewChange`类型的`Payload`，和对消息的签名，    
`viewchange.go` 257行  `sendNewView`方法中获得`NewView`类型的`Payload`，  
`pbft-core.go`  1222行  `fetchRequestBatches`方法中获得`FetchRequestBatch`类型的`Payload`，  
然后都是经过一系列的调用后经过`wrapMessage`对`Payload`进行进一步的包装并给出消息类型`Type`。

