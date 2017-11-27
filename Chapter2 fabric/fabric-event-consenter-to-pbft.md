## 摘要

fabric中用了`obcBacth`来实现`Consenter`接口, 但没有直接实现接受消息的功能, 而是继承了pbft中定义的`externalEventReceiver`. `externalEventReceiver`会把消息进行相应的封装, 然后传给自己的成员变量`manager`, 该变量在`utils/event`中定义和实现. 在`obcBatch`初始化时会启动`manager`, `manager`启动后便会持续调用`obcBatch`的`func ProcessEvent()`, 该函数会对消息进行类型识别, 并调用pbft进行相应操作.

即: Consenter(externalEventReceiver)(封装) >> event.Manager >> Consenter(obcBacth)(初步解析) >> pbft

共识机制内部生成的事件也会交与`Consenter`处理.

## 过程

`consenter`是`engine`的成员变量, 在启动程序之后, `engine`便会启动线程持续地接受来自`peer`的消息, 并调用`func consenter.RecvMsg()`进行处理.

`Consenter`是在`consensus.go`定义的接口, 通过`engine.consenter = controller.NewConsenter(engine.helper)`初始化, 经过一系列调用, 可以追溯到, 在pbft中, Consenter接口是`type obcBatch strcut`来实现的. 但是`obcBatch`自身并没有实现`func RecvMsg()`, 此函数是继承自`externalEventReceiver`. 此结构体定义在`consensus/pbft/external.go`.
```go
type externalEventReceiver struct {
	manager events.Manager
}
...
// RecvMsg is called by the stack when a new message is received
func (eer *externalEventReceiver) RecvMsg(ocMsg *pb.Message, senderHandle *pb.PeerID) error {
	eer.manager.Queue() <- batchMessageEvent{
		msg:    ocMsg,
		sender: senderHandle,
	}
	return nil
}
...
// Executed is called whenever Execute completes, no-op for noops as it uses the legacy synchronous api
func (eer *externalEventReceiver) Executed(tag interface{}) {
	eer.manager.Queue() <- executedEvent{tag}
}

// Committed is called whenever Commit completes, no-op for noops as it uses the legacy synchronous api
func (eer *externalEventReceiver) Committed(tag interface{}, target *pb.BlockchainInfo) {
	eer.manager.Queue() <- committedEvent{tag, target}
}
...
```
可以看到, 再收到消息后, `externalEventReceiver`又把它传给了`events.Manager`. 在`externalEventReceiver`中还定义了`Executed`, `Committed`, `RolledBack`等操作, 但从代码可以看出, 它并没有对消息进行特别的处理, 只是根据调用方法的不同, 分别把消息封装了一下, 然后全部传给`eer.manager.Queue()`. 查找`Executed`等事件可以发现, 这些事件都是共识机制内部发起的, 用于内部通信.

`events.Manager`是一个接口, 定义在`consensus/util/events/events.go`, 不在pbft内部.
```go
type Manager interface {
	Inject(Event)         // A temporary interface to allow the event manager thread to skip the queue
	Queue() chan<- Event  // Get a write-only reference to the queue, to submit events
	SetReceiver(Receiver) // Set the target to route events to
	Start()               // Starts the Manager thread TODO, these thread management things should probably go away
	Halt()                // Stops the Manager thread
}
```
其用`type managerImpl struct`实现该接口, 
```go
type managerImpl struct {
	threaded
	receiver Receiver
	events   chan Event
}
```
在`managerImpl`的成员函数`func (em *managerImpl) Queue() chan<- Event`中, `managerImpl`返回了`events`, 即, 传往`Manager`的消息会最终传给`events`.

回到`Consenter`. 在初始化`obcBatch`在`func newObcBatch()`中, 会将`obcBatch`的一个成员变量`manager`初始化成`managerImpl`, 但稍后又用`manager`初始化了`op.externalEventReceiver.manager`. 本身`obcBatch`是继承了`externalEventReceiver`, 就算没有定义自己的`manager`仍会从`externalEventReceiver`继承到`manager`. 代码中注释介绍:
```go
type obcBatch struct {
	...
	externalEventReceiver
	...
	manager events.Manager // TODO, remove eventually, the event manager
	...
}
...
func newObcBatch(id uint64, config *viper.Viper, stack consensus.Stack) *obcBatch {
	...
	op.manager = events.NewManagerImpl() // TODO, this is hacky, eventually rip it out
	op.manager.SetReceiver(op)
	etf := events.NewTimerFactoryImpl(op.manager)
	op.pbft = newPbftCore(id, config, op, etf)
	op.manager.Start()
	...
	op.externalEventReceiver.manager = op.manager
	...
}
```
所以反复的赋值, 这种机制属于geth本身未修复一个bug, 对外应该只需要`externalEventReceiver`即可. 具体原因未知.

`obcBatch`把自己设为`manager`的`receiver`, 稍后启动了`manager`, 查看`Manager`的代码会发现, `manager`启动后会开启一个新的线程持续读取`events`中的信息, 并调用`receiver`的`func ProcessEvent()`:
```go
func SendEvent(receiver Receiver, event Event) {
	next := event
	for {
		// If an event returns something non-nil, then process it as a new event
		next = receiver.ProcessEvent(next)
		if next == nil {
			break
		}
	}
}
```
`obcBatch`实现了`func ProcessEvent()`, 在该函数中, `obcBatch`会判断`event`的类型, 并执行对应的函数, 
```go
	switch et := event.(type) {
	case batchMessageEvent:
		ocMsg := et
		return op.processMessage(ocMsg.msg, ocMsg.sender)
	...
	case stateUpdatedEvent:
		// When the state is updated, clear any outstanding requests, they may have been processed while we were gone
		op.reqStore = newRequestStore()
		return op.pbft.ProcessEvent(event)
	default:
		return op.pbft.ProcessEvent(event)
```
可以看到, 针对特定的事件类型, `obcBatch`对其进行了处理, 并将某些事件直接传递给pbft来处理. 至此, 消息就已经从`consenter`传送到了`pbft`.