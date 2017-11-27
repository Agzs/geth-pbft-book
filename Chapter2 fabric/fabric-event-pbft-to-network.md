## 摘要

geth对消息传递的接口进行了多次封装. pbft内部使用broacdaster发送消息, broadcaster用成员变量communicator发送消息, communicator本质上是consensus.helper, helper使用了成员变量coordinator发送消息, coordinator是定义在`core/peer`中的接口, 其由peer.Impl来实现. 即最终会通过peer.Impl把消息交给网络层.

pbft >> pbft.broadcaster >> pbft.communicator >> consensus.helper >> consensus.helper.coordinator >> peer.Impl

## 过程

在pbft中定义了类型`broadcaster`, 其实现了`func Unicast()`和`func Broadcast()`等函数, 内部调用了`func send()`, 最终会向自己内部的`msgChans`发送消息.
```go
type broadcaster struct {
	comm communicator

	f                int
	broadcastTimeout time.Duration
	msgChans         map[uint64]chan *sendRequest
	closed           sync.WaitGroup
	closedCh         chan struct{}
}
```

```go
func (b *broadcaster) send(msg *pb.Message, dest *uint64) error {
	...
	for i := range b.msgChans {
		b.unicastOne(msg, i, wait)
	}
	...
}
```

`msgChans`是在`func newBroadcaster()`自己定义的, 在该函数中, 会生成N个channel, N是网络总共节点(replica)的个数. 同时在该函数中还会启动N个线程, 运行`func drainer()`负责从msgChans中读取消息, 然后分发给节点. `func drainer()`会进一步调用`func drainerSend()`, 在`func drainerSend()`中, `broadcaster`调用成员变量`comm communicator`的方法来向外部发送消息. `communicator`是在`broadcaster`初始化时同时初始化的, 而`broadcaster`是在`obcBatch`初始化时初始化的. 在`broadcaster`初始化过程中, 将一个类型为`consensus.Stack`的数据传给`broadcaster`作为它的`communicator`. 这个数据在fabric启动过程中已经提到, 本质上是`consensus`中定义的`engine.helper`. 所以本质上它是通过`helper`的`func Unicast()`和`func Broadcast()`来实现消息传递的.

```go
// Broadcast sends a message to all validating peers
func (h *Helper) Broadcast(msg *pb.Message, peerType pb.PeerEndpoint_Type) error {
	errors := h.coordinator.Broadcast(msg, peerType)
	if len(errors) > 0 {
		return fmt.Errorf("Couldn't broadcast successfully")
	}
	return nil
}
// Unicast sends a message to a specified receiver
func (h *Helper) Unicast(msg *pb.Message, receiverHandle *pb.PeerID) error {
	return h.coordinator.Unicast(msg, receiverHandle)
}
```

可以看到, `helper`其实也是调用了成员变量`coordinator`的函数来实现通信. `coordinator`也是一个接口, 实际上是`peer.MessageHandlerCoordinator`, 定义在`peer`中. `core/peer/peer.go`中`Impl`类型实现了该接口, 同时也实现了`func Unicast()`和`func Broadcast()`等函数. 即, 消息最终是通过调用`peer.Impl`的函数发送的. 接下来的过程不再深究.