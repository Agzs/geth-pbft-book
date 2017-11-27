`Stack`是`consensus`里一个主要的接口, 包含了`consensus.go`里定义的除自身和`Consenter`之外的所有接口.

在`pbft`内部, 也定义了一个与此对应的接口`innerStack`, 用于内部使用.

对于`Stack`里一些方法的介绍在前面各部分已经提到了一部分, 这里综合地描述下`Stack`的各个部分.

> Stack is the set of stack-facing methods available to the consensus plugin

```go
type Stack interface {
	NetworkStack
	SecurityUtils
	Executor
	LegacyExecutor
	LedgerManager
	ReadOnlyLedger
	StatePersistor
}
```

## Vars
虽然在注释中地提到了在`consensus/helper/handler.go`中定义的`ConsensusHandler`也实现了`Stack`, 但是查找代码会发现, 其实只有`consensus/helper/helper.go`中定义的`Helper`实现了`Stack`. 所以下面只会讨论`Helper`.

> Helper contains the reference to the peer's MessageHandlerCoordinator

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
### `consenter`
`consenter`成员变量的类型为`Consenter`, 即`consensus`里定义的`Consenter`接口. 注意到, `Stack`并不包括`Consenter`.
```go
// ExecutionConsumer allows callbacks from asycnhronous execution and statetransfer
type ExecutionConsumer interface {
	Executed(tag interface{})                                // Called whenever Execute completes
	Committed(tag interface{}, target *pb.BlockchainInfo)    // Called whenever Commit completes
	RolledBack(tag interface{})                              // Called whenever a Rollback completes
	StateUpdated(tag interface{}, target *pb.BlockchainInfo) // Called when state transfer completes, if target is nil, this indicates a failure and a new target should be supplied
}

// Consenter is used to receive messages from the network
// Every consensus plugin needs to implement this interface
type Consenter interface {
	RecvMsg(msg *pb.Message, senderHandle *pb.PeerID) error // Called serially with incoming messages from gRPC
	ExecutionConsumer
}
```

在`Engine`的初始化过程中, 会先创建`Helper`, 然后用`Helper`作为参数创建`obcBatch`(即`Consenter`的实现), 然后再把`obcBatch`赋给`Helper`, 整个过程下来, `Helper`和`Consenter`两者会互相包含对方作为成员变量.

`Helper`中仅使用`consenter`来实现`ExecutionConsumer`接口. 在`ExecutionConsumer`已经提到. 这些方法将会在`executor`中被使用.

### `coordinator`
`coordinator`类型为`peer.MessageHandlerCoordinator`, 定义在`peer`中, 是初始化`engine`时的唯一参数, 也可以说是和节点联系的途径. `engine`使用它获得`PeerEndpoint`(本节点), 然后将它传给`Helper`用于初始化, 自己并不保存这个变量. `Helper`会保存该变量, 并用它来实现了`NetworkStack`里的功能, 包括获取节点信息, 广播, 单播, 另外还包括获取区块链长度.
```go
h.coordinator.GetPeerEndpoint()
h.coordinator.GetPeers()
h.coordinator.Broadcast(msg, peerType)
h.coordinator.Unicast(msg, receiverHandle)
h.coordinator.GetBlockchainSize()
```
`ConsensusHandler`中也保存了该类型变量, 但并未使用.

该变量也会被传给`executor`, 将在后面提到.

### `secOn`
一个类型为`bool`的简单变量, 判断是否对消息进行加密(Sign&Verify), 由配置信息决定取值, 且在程序运行时不再改变.

### `valid`
类型为`bool`. 用于标记是否当前状态是否为最新.
> Whether we believe the state is up to date

用于实现`LedgerManager`接口, 在`LedgerManager`中已经提到了使用情况.

在`engine`执行`func ProcessTransactionMsg()`和`Helper`执行`func UpdateState()`时会检查`valid`的取值.

### `secHelper`
用于实现`SecurityUtils`接口, 类型为`crypto.Peer`, 通过调用`coordinator.GetSecHelper()`生成. 在消息传递前会进行加密, 接收后进行解密.

实际上, 在`pbft`中并没有直接使用`SecurityUtils`接口, 而是进行了封装, 但主要还是`SecurityUtils`接口部分.

### `curBatch` & `curBatchErrs`
类型分别为`[]*pb.Transaction`和`[]*pb.TransactionResult`, 储存Tx和Tx的执行结果, 注释中提到, 这两个变量将被删去.
```go
curBatch     []*pb.Transaction       // TODO, remove after issue 579
curBatchErrs []*pb.TransactionResult // TODO, remove after issue 579
```
这两个变量将会在记录产生的Tx, 然后通过`func CommitTxBatch()`提交.

### `persist.Helper`
`Help`继承了`persist.Helper`, 用于实现`StatePersistor`接口. 暂不关注.

### `executor`
类型为`consensus.Executor`, 用于实现`Executor`接口. 通过`executor.NewImpl(h, h, mhc)`初始化, 这里, `h`是指`Helper`自身, `mhc`则是前面提到的`peer.MessageHandlerCoordinator`. 这里`mhc`是作为`statetransfer.PartialStack`使用的, `statetransfer`定义在`core/peer/statetransfer/statetransfer.go`, 用来转换状态, 具体地, 在`executor`里除了调用外`func Start()`, `func Halt()`只是在收到`stateUpdateEvent`后调用`func SyncToTarget()`来将区块链更新到目标状态.

前面已经提到过`executor`的执行模式, 即把所有消息都发送给`event.Manager`后经`event.Manager`调用`func ProcessEvent()`来执行. `executor`也不具体规定执行动作, 而只是把`Helper`里的方法(即接口`ExecutionConsumer`和`LegacyExecutor`)封装起来, 实际动作是在后面那两个接口里定义的.

## func
前面围绕着各个变量已经提到了一部分函数, 除此之外还定义了一些函数用于实现接口.

### `Executor`
针对这个接口, 主要函数是通过`executor`实现的, 特别的, `Executor`定义了`func Start()`, `func Halt()`, 而这两个函数对于`Helper`来说是无意义的, 所以, 仅仅为了实现`Executor`, `Helper`中定义了两个空函数, 这样做只是为了满足编程语言规范. 另外, `executor`实际实现了`Executor`接口, 所以也定义了这两个函数, 在这两个函数中分别启动和停止`event.Manager`和`statetransfer`. 而`executor`本身的启动, 是在`func setConsenter()`里进行的. (由于`Helper`的初始化后紧接着就执行了`func setConsenter()`, 所以也可以理解成`Helper`初始化时边启动了`executor`.

### `ReadOnlyLedger`
这些部分是由`Helper`和`Ledger`等交互实现的, 在`ReadOnlyLedger`里已经提到, 不再重复.