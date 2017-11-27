# Blockchain
## `ReadOnlyLedger`
> ReadOnlyLedger is used for interrogating the blockchain.

```go
type ReadOnlyLedger interface {
	GetBlock(id uint64) (block *pb.Block, err error)
	GetBlockchainSize() uint64
	GetBlockchainInfo() *pb.BlockchainInfo
	GetBlockchainInfoBlob() []byte
	GetBlockHeadMetadata() ([]byte, error)
}
```
在fabric中, 储存交易的最高层的数据结构称作`Ledger`, `blockchain`是定义在`Ledger`之内的, 同时也是`Ledger`私有的, 不暴露给外部使用. 因此, 对于`blockchain`的操作, 都是经过`Ledger`.

### Usage
和前面介绍的接口一样, `ReadOnlyLedger`也只作为`Stack`的一部分出现.

### Implement
在`Helper`中, 每次对`ReadOnlyLedger`内几个函数的实现, 都是先通过调用`func ledger.GetLedger()`获得`ledger`, 然后执行对应的方法.
```go
// GetBlock returns a block from the chain
func (h *Helper) GetBlock(blockNumber uint64) (block *pb.Block, err error) {
	ledger, err := ledger.GetLedger()
	if err != nil {
		return nil, fmt.Errorf("Failed to get the ledger :%v", err)
	}
	return ledger.GetBlockByNumber(blockNumber)
}
```
查看`func GetLedger()`其实可以发现, 它也并没有每次都生成一个`Ledger`, 而是只生成一次, 以后的都会返回同一个.
```go
// GetLedger - gives a reference to a 'singleton' ledger
func GetLedger() (*Ledger, error) {
	once.Do(func() {
		ledger, ledgerError = GetNewLedger()
	})
	return ledger, ledgerError
}
```
`Ledger`内部自己是通过调用`blockchain`的方法来实现的.
```go
func (ledger *Ledger) GetBlockByNumber(blockNumber uint64) (*protos.Block, error) {
	if blockNumber >= ledger.GetBlockchainSize() {
		return nil, ErrOutOfBounds
	}
	return ledger.blockchain.getBlock(blockNumber)
}
```
`blockchain`内则通过操作数据库来实现.

### Summary
在fabric中, `blockchain`被封装在了`Ledger`内, `ReadOnlyLedger`定义了对Blockchain的读操作, `Helper`使用`Ledger`对其进行了实现, 而`Ledger`内部则是调用了`blockchain`的对应方法, `blockchain`则是通过使用`db`进行了实现.

## `LegacyExecutor`
> LegacyExecutor is used to invoke transactions, potentially modifying the backing ledger

```go
type LegacyExecutor interface {
	BeginTxBatch(id interface{}) error
	ExecTxs(id interface{}, txs []*pb.Transaction) ([]byte, error)
	CommitTxBatch(id interface{}, metadata []byte) (*pb.Block, error)
	RollbackTxBatch(id interface{}) error
	PreviewCommitTxBatch(id interface{}, metadata []byte) ([]byte, error)
}
```
和`ReadOnlyLedger`类似, 只不过`LegacyExecutor`内定义的是写操作.

### Usage
- 作为`Stack`的一部分. 此处和`ReadOnlyLedger`类似.

- 作为`consensus/executor.PartialStack`接口定义的一部分.
```go
// PartialStack contains the ledger features required by the executor.Coordinator
type PartialStack interface {
	consensus.LegacyExecutor
	GetBlockchainInfo() *pb.BlockchainInfo
}
```
`PartialStack`只在`executor`内部被作为`type coordinatorImpl struct`的成员变量来使用. `executor`是作为`Helper`的一部分来使用, 用来处理实际Tx的执行动作. 前面提到`obcBatch`使用了`event.Manager`模块进行消息接收处理, 其实`executor`也注册了一个`event.Manager`用来接收执行后的消息. 就是说, 当`obcEvent`接收到来自`pbft`的消息后(比如`executedEvent`), `obcEvent`会执行相应动作(比如`Commit`), 这个动作是在`Helper`里定义的, 但是是通过`executor`来实现的, `executor`会把对应时间发给自己的`EventManager`, `EventManager`再将消息传回`executor`, `executor`收到消息后进行执行. 执行动作就会用到我们刚才提到的接口.

### Implement
虽然这里也有两类使用, 但是Usage#2中, `PartialStack`接口也是用`Helper`来实现的, 所以也就只有在`Helper`处的一处实现.

在`Helper`中的实现, 一部分也是直接通过与`Ledger`交互来实现的. 如:
```go
func (h *Helper) BeginTxBatch(id interface{}) error {
	ledger, err := ledger.GetLedger()
	if err != nil {
		return fmt.Errorf("Failed to get the ledger: %v", err)
	}
	if err := ledger.BeginTxBatch(id); err != nil {
		return fmt.Errorf("Failed to begin transaction with the ledger: %v", err)
	}
	...
}
```
而对于`func ExecTxs()`, 则是通过`chaincode`.
```go
func (h *Helper) ExecTxs(id interface{}, txs []*pb.Transaction) ([]byte, error) {
	...
	succeededTxs, res, ccevents, txerrs, err := chaincode.ExecuteTransactions(context.Background(), chaincode.DefaultChain, txs)
	...
}
```
 
### Summary
`LegacyExecutor`用来对`blockchain`进行写操作, 实际实现只有一处, 在`Helper`中, 另一处在`executor`里虽作为一部分定义了一个新的接口, 但该接口仍是通过`Helper`来初始化的. 在`Helper`中, 除`func ExecTxs()`是和`chaincode`交互实现的, 其他仍是和`Ledger`交互. `executor`的存在, 主要是为了方便收集来自`obcBatch`关于对`blockchain`写操作的消息, 集中调用`Helper`进行执行.

## `Executor`
> Executor is intended to eventually supplant the old Executor interface. The problem with invoking the calls directly above, is that they must be coordinated with state transfer, to eliminate possible races and ledger corruption.

```go
type Executor interface {
	Start()                                                                     // Bring up the resources needed to use this interface
	Halt()                                                                      // Tear down the resources needed to use this interface
	Execute(tag interface{}, txs []*pb.Transaction)                             // Executes a set of transactions, this may be called in succession
	Commit(tag interface{}, metadata []byte)                                    // Commits whatever transactions have been executed
	Rollback(tag interface{})                                                   // Rolls back whatever transactions have been executed
	UpdateState(tag interface{}, target *pb.BlockchainInfo, peers []*pb.PeerID) // Attempts to synchronize state to a particular target, implicitly calls rollback if needed
}
```
`Executor`是用来执行tx的, 其中也会包含执行后的动作(调用`func Commited()`等).

### Usage
`Executor`也被定义为`Stack`的一部分, 在`Helper`内进行了实现, 实际上是`Helper`用`executor`来实现的, `Helper`只是进行了封装, `Helper`这些函数, 在`func obcBatch.ProcessEvent()`中被调用(并非全部使用).

### Implement
`executor`对这些方法只是简单地将数据封装成一个新的时间, 交给`event.Manager`, 然后在`func ProcessEvent()`中集中处理.
```go
func (co *coordinatorImpl) Commit(tag interface{}, metadata []byte) {
	co.manager.Queue() <- commitEvent{tag, metadata}
}
```
```go
func (co *coordinatorImpl) ProcessEvent(event events.Event) events.Event {
	switch et := event.(type) {
	case executeEvent:
		...
		co.rawExecutor.ExecTxs(co, et.txs)
		co.consumer.Executed(et.tag)
	case commitEvent:
		...
		_, err := co.rawExecutor.CommitTxBatch(co, et.metadata)
		...
		info := co.rawExecutor.GetBlockchainInfo()
		...
		co.consumer.Committed(et.tag, info)
	case rollbackEvent:
		...
	case stateUpdateEvent:
		...
	default:
		logger.Errorf("Unknown event type %s", et)
	}

	return nil
}
```
注意到`rawExecutor`, 它其实就是实现了`LegacyExecutor`的接口(实际上是`PartialStack`), 后面又调用了`consumer`, 即Event中介绍的`ExecutionConsumer`, 从这里就可以看出, `Executor`其实就是把两个接口内的方法又进行了封装.

### Summary
`Executor`作为`Stack`的一部分以`Helper`实现, 本质上是由`executor.coordinatorImpl`来实现的. 在内容上, 就是将`LegacyExecutor`和`ExecutionConsumer`接口封装到一块.

## `LedgerManager`
> LedgerManager is used to manipulate the state of the ledger

```go
type LedgerManager interface {
	InvalidateState() // Invalidate informs the ledger that it is out of date and should reject queries
	ValidateState()   // Validate informs the ledger that it is back up to date and should resume replying to queries
}
```
`LedgerManager`内只定义了两个方法, 而且还都没有参数, 就是**设置**当前的`Ledger`是否是最新的.

### Usage
只作为`Stack`的一部分使用. 两个函数被`pbft`进一步封装成了`innerStack`的函数, 会在需要临时禁用`Ledger`的时候(`func stateTransfer()`等)处或检测到状态过期的时候调用`func InvalidateState()`, 会在收到`stateUpdatedEvent`消息后调用`func ValidateState()`重新启用`Ledger`.

### Implement
实现很简单, 在`Helper`中保存了一个`bool`变量, 分别置为false, true.

### Summary
`LedgerManager`也是`Stack`的一部分, 用于设置`Ledger`的状态是否最新/合法, 会在状态转换前或检测到失效后置false, 收到`stateUpdatedEvent`后重新置true.