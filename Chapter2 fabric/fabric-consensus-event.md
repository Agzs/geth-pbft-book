# Event
## `ExecutionConsumer`
> ExecutionConsumer allows callbacks from asycnhronous execution and statetransfer.

```go
type ExecutionConsumer interface {
	Executed(tag interface{})                                // Called whenever Execute completes
	Committed(tag interface{}, target *pb.BlockchainInfo)    // Called whenever Commit completes
	RolledBack(tag interface{})                              // Called whenever a Rollback completes
	StateUpdated(tag interface{}, target *pb.BlockchainInfo) // Called when state transfer completes, if target is nil, this indicates a failure and a new target should be supplied
}
```
`ExecutionConsumer`主要定义了Tx被处理之后的操作(从函数名可以看出, 都是过去式).

### Usage
- 作为`consensus.Consenter`接口的一部分.
```go
type Consenter interface {
	RecvMsg(msg *pb.Message, senderHandle *pb.PeerID) error // Called serially with incoming messages from gRPC
	ExecutionConsumer
}
```
即实现`Consenter`的结构也同时需要实现`ExecutionConsumer`. 具体被调用情况见下文对`Consenter`的介绍.

- 作为`consensus/executor.coordinatorImpl`的成员变量`consumer`.
```go
type coordinatorImpl struct {
	manager         events.Manager              // Maintains event thread and sends events to the coordinator
	rawExecutor     PartialStack                // Does the real interaction with the ledger
	consumer        consensus.ExecutionConsumer // The consumer of this coordinator which receives the callbacks
	stc             statetransfer.Coordinator   // State transfer instance
	batchInProgress bool                        // Are we mid execution batch
	skipInProgress  bool                        // Are we mid state transfer
}
```
`executor.coordinatorImpl`是用来处理事件的, 定义了`func ProcessEvent()`, 在执行完一个事件之后, 该函数会调用`consumer`的对应的各函数.

### Implement
程序中没有单独实现的`ExecutionConsumer`. 即便对于Usage #2, 该成员变量也是用`consensus.helper`来初始化的. 而`consensus.helper`对该接口的实现也仅仅是调用`helper`的成员变量`consenter`. 所以, 代码中仅有实现`Consenter`的类型对`ExecutionConsumer`进行了实现. 对于`Consenter`的实现, 在`Consenter`部分介绍.
```go
// Executed is called whenever Execute completes
func (h *Helper) Executed(tag interface{}) {
	if h.consenter != nil {
		h.consenter.Executed(tag)
	}
}
```

### Summary
程序中有两处使用`ExecutionConsumer`, 一处作为`Consenter`接口的一部分, 另一处作为`consensus/executor.coordinatorImpl`的成员变量. 两者都是由通过`Consenter`实现的, 所以不必单独考虑`ExecutionConsumer`的实现, 只考虑`Consenter`中对应函数的实现即可. 事实上, 该接口单独定义出来只是为了把功能区分出来, 其实完全可以将该接口定义在`Consenter`里, 因为事实上程序使用的也只有`Consenter`.

## `Consenter`
> Consenter is used to receive messages from the network. Every consensus plugin needs to implement this interface

```go
type Consenter interface {
	RecvMsg(msg *pb.Message, senderHandle *pb.PeerID) error // Called serially with incoming messages from gRPC
	ExecutionConsumer
}
```
在fabirc/event中已经提到, `Consenter`是`consensus`中接收事件的接口. 此外还负责`consense`内部的通信功能.

### Usage
- 作为`consensus/controller`的全局变量.
```go
var consenter consensus.Consenter
```
`controller`虽然保存了一个全局变量, 但是, 该变量为私有变量, 不能被外部访问, 但是`controller`又没有对该变量进行操作, 所以该变量也是多余的. 并没有什么用. 另外, 在`controller`中, 定义了`func NewConsenter(stack consensus.Stack) consensus.Consenter`, 该函数会根据配置信息选择何种共识机制. 此函数会在初始化`engine`时被使用. 除此之外`controller`内也没有其他功能了.

- 作为`consensus/helper.EngineImpl`的成员变量.
```go
type EngineImpl struct {
	consenter    consensus.Consenter
	helper       *Helper
	peerEndpoint *pb.PeerEndpoint
	consensusFan *util.MessageFan
}
```
在fabric/event中提到了, 在`EngineImpl`的初始化函数中会开启一个新的线程, 不断地从网络获取消息. 该过程会重复调用`func consenter.RecvMsg()`. 另外在`func EngineImpl.ProcessTransactionMsg()`中, 在执行完一个tx消息后, 会开始等待接收一个返回消息. 特别的, 代码中也有注释, 讨论其必要性.
> TODO, do we want to put these requests into a queue? This will block until the consenter gets around to handling the message, but it also provides some natural feedback to the REST API to determine how long it takes to queue messages

```go
err := eng.consenter.RecvMsg(msg, eng.peerEndpoint.ID)
```
有些不解的是, 该函数的调用的`sender`参数是来自`engine`的一个成员变量, 但观察代码暂时只发现对于该成员变量只有一处赋值. 这个过程可能会在详细分析`engine`的时候弄清楚.

- 作为`consensus/helper.Helper`的成员变量.
```go
type Helper struct {
	consenter    consensus.Consenter
	coordinator  peer.MessageHandlerCoordinator
	secOn        bool
	valid        bool // Whether we believe the state is up to date
	secHelper    crypto.Peer
	curBatch     []*pb.Transaction       // TODO, remove after issue 579
	curBatchErrs []*pb.TransactionResult // TODO, remove after issue 579
	persist.Helper

	executor consensus.Executor
}
```
在`ExecutionConsumer`已经提到, `Helper`通过自己成员变量`consenter`实现了`ExecutionConsumer`接口, 事实上也仅有这一个用处. 当`Consenter`以`ExecutionConsumer`的身份作为`consensus/executor.coordinatorImpl`的成员变量时, 被用在`func (co *coordinatorImpl) ProcessEvent(event events.Event) events.Event`. 在`fabric/event`里已经提到过, 这里会根据事件类型进行不同的处理, 其中就包括有些动作处理完成之后对`func Executed()`等的调用.

- 作为`consensus/pbft`的全局变量.
```go
var pluginInstance consensus.Consenter // singleton service
```
前面提到了, `controller`在`engine`初始化时会调用`func pbft.GetPlugin()`生成一个`Consenter`作为`engine`的成员变量.
```go
// GetPlugin returns the handle to the Consenter singleton
func GetPlugin(c consensus.Stack) consensus.Consenter {
	if pluginInstance == nil {
		pluginInstance = New(c)
	}
	return pluginInstance
}
```
可以看到, `pbft`只会生成一个`consensus.Consenter`, 以后的调用都会返回同一个`Consenter`. `pbft`自身并没有直接使用`pluginInstance`.

### Implement
`Helper`的`Consenter`来自于`engine`, `engine`的`Consenter`来自`controller`, `controller`的来自于`pbft`. `pbft`使用`func New()`构建, 如下所示.
```go
func New(stack consensus.Stack) consensus.Consenter {
	handle, _, _ := stack.GetNetworkHandles()
	id, _ := getValidatorID(handle)

	switch strings.ToLower(config.GetString("general.mode")) {
	case "batch":
		return newObcBatch(id, config, stack)
	default:
		panic(fmt.Errorf("Invalid PBFT mode: %s", config.GetString("general.mode")))
	}
}
```
可以看到, `Consenter`接口的实现为`obcBatch`, 来自于`func newObcBatch()`. 该函数定义在`consensus/pbft/batch.go`. 但正如`fabric/event`中提到的, `obcBatch`并没有自己实现这些函数, 而是继承自`externalEventReceiver`. 该类型定义在`consensus/pbft/external.go`, 该类型通过和`events.Manager`交互实现了接口中定义的五个函数.
```go
// RecvMsg is called by the stack when a new message is received
func (eer *externalEventReceiver) RecvMsg(ocMsg *pb.Message, senderHandle *pb.PeerID) error {
	eer.manager.Queue() <- batchMessageEvent{
		msg:    ocMsg,
		sender: senderHandle,
	}
	return nil
}

// Executed is called whenever Execute completes, no-op for noops as it uses the legacy synchronous api
func (eer *externalEventReceiver) Executed(tag interface{}) {
	eer.manager.Queue() <- executedEvent{tag}
}

// Committed is called whenever Commit completes, no-op for noops as it uses the legacy synchronous api
func (eer *externalEventReceiver) Committed(tag interface{}, target *pb.BlockchainInfo) {
	eer.manager.Queue() <- committedEvent{tag, target}
}

// RolledBack is called whenever a Rollback completes, no-op for noops as it uses the legacy synchronous api
func (eer *externalEventReceiver) RolledBack(tag interface{}) {
	eer.manager.Queue() <- rolledBackEvent{}
}

// StateUpdated is a signal from the stack that it has fast-forwarded its state
func (eer *externalEventReceiver) StateUpdated(tag interface{}, target *pb.BlockchainInfo) {
	eer.manager.Queue() <- stateUpdatedEvent{
		chkpt:  tag.(*checkpointMessage),
		target: target,
	}
}
```

### Summary
可以看到, 对于`Consenter`的使用分为两部分, 分别为`engine`使用`func RevMsg()`来接收消息以及`Helper`调用`ExecutionConsumer`部分的函数来对接收到的消息进行处理. 其余两处的调用仅是对数据的一些储存和传递, 并没有直接使用. `Consenter`接口由`obcBatch`来实现, 但它自己也没有实现这些功能, 而是继承了`externalEventReceiver`, `externalEventReceiver`通过和`event.Manager`的直接交互来对接口进行了实现.