### ! 本部分内容有遗漏, 待补充

geth启动后, 经过一系列过程, 在`cmd/utils/flags.go`中会通过`func RegisterEthService(stack *node.Node, cfg *eth.Config)`调用`eth.New()`生成`type Ethereum struct`实例, 其中包含了关于协议的各种信息, 包括`consensus`, `miner`, `blockchain`等.

```go
// Ethereum implements the Ethereum full node service.
type Ethereum struct {
	chainConfig *params.ChainConfig
	// Channel for shutting down the service
	shutdownChan  chan bool // Channel for shutting down the ethereum
	stopDbUpgrade func()    // stop chain db sequential key upgrade
	// Handlers
	txPool          *core.TxPool
	txMu            sync.Mutex
	blockchain      *core.BlockChain
	protocolManager *ProtocolManager
	lesServer       LesServer
	// DB interfaces
	chainDb ethdb.Database // Block chain database

	eventMux       *event.TypeMux
	engine         consensus.Engine
	accountManager *accounts.Manager

	ApiBackend *EthApiBackend

	miner     *miner.Miner
	gasPrice  *big.Int
	etherbase common.Address

	networkId     uint64
	netRPCService *ethapi.PublicNetAPI

	lock sync.RWMutex // Protects the variadic fields (e.g. gas price and etherbase)
}
```
其中`engine`储存了`consensus.Engine`实例, 在`New()`中由`CreateConsensusEngine(ctx, config, chainConfig, chainDb)`生成.
```go
	eth := &Ethereum{
		chainDb:        chainDb,
		chainConfig:    chainConfig,
		eventMux:       ctx.EventMux,
		accountManager: ctx.AccountManager,
		engine:         CreateConsensusEngine(ctx, config, chainConfig, chainDb),
		shutdownChan:   make(chan bool),
		stopDbUpgrade:  stopDbUpgrade,
		networkId:      config.NetworkId,
		gasPrice:       config.GasPrice,
		etherbase:      config.Etherbase,
	}
```
在`New()`接下来的部分, 有以下命令与`engine`相关.
```go
eth.blockchain, err = core.NewBlockChain(chainDb, eth.chainConfig, eth.engine, eth.eventMux, vmConfig)
```
```go
eth.protocolManager, err = NewProtocolManager(eth.chainConfig, config.SyncMode, config.NetworkId, maxPeers, eth.eventMux, eth.txPool, eth.engine, eth.blockchain, chainDb)
```
```go
eth.miner = miner.New(eth, eth.chainConfig, eth.EventMux(), eth.engine)
```

## blockchain
在`core/blockchain.go`描述了`blockchain`.
`BlockChain`结构体定义如下
```go
type BlockChain struct {
	config *params.ChainConfig // chain & network configuration

	hc           *HeaderChain
	chainDb      ethdb.Database
	eventMux     *event.TypeMux
	genesisBlock *types.Block

	mu      sync.RWMutex // global mutex for locking chain operations
	chainmu sync.RWMutex // blockchain insertion lock
	procmu  sync.RWMutex // block processor lock

	checkpoint       int          // checkpoint counts towards the new checkpoint
	currentBlock     *types.Block // Current head of the block chain
	currentFastBlock *types.Block // Current head of the fast-sync chain (may be above the block chain!)

	stateCache   state.Database // State database to reuse between imports (contains state cache)
	bodyCache    *lru.Cache     // Cache for the most recent block bodies
	bodyRLPCache *lru.Cache     // Cache for the most recent block bodies in RLP encoded format
	blockCache   *lru.Cache     // Cache for the most recent entire blocks
	futureBlocks *lru.Cache     // future blocks are blocks added for later processing

	quit    chan struct{} // blockchain quit channel
	running int32         // running must be called atomically
	// procInterrupt must be atomically called
	procInterrupt int32          // interrupt signaler for block processing
	wg            sync.WaitGroup // chain processing wait group for shutting down

	engine    consensus.Engine
	processor Processor // block processor interface
	validator Validator // block and state validator interface
	vmConfig  vm.Config

	badBlocks *lru.Cache // Bad block cache
}
```
注意到, 里面包含
```go
	engine    consensus.Engine
	processor Processor // block processor interface
	validator Validator // block and state validator interface
```
这些是与共识机制相关的. `BlockChain`由`func NewBlockChain(chainDb ethdb.Database, config *params.ChainConfig, engine consensus.Engine, mux *event.TypeMux, vmConfig vm.Config) (*BlockChain, error)`生成. 在内部, 除了将从`eth`获得的`engine`保存到自己的结构体内, 还通过调用
```go
	bc.SetValidator(NewBlockValidator(config, bc, engine))
	bc.SetProcessor(NewStateProcessor(config, bc, engine))
```
定义了`processor`和`validator`. 

`func NewBlockValidator()`在`core/block_validator.go`中定义. 在后面的写块等处, 使用了形如`bc.Validator().ValidateBody(block)`的方式来验证块的合法性, `BlockValidator`使用了`consensus.VerifyUncles()`.
```go
// NewBlockValidator returns a new block validator which is safe for re-use
func NewBlockValidator(config *params.ChainConfig, blockchain *BlockChain, engine consensus.Engine) *BlockValidator {
	validator := &BlockValidator{
		config: config,
		engine: engine,
		bc:     blockchain,
	}
	return validator
}
```

`NewStateProcessor()`在`core/state_processor.go`中定义, `StateProcessor`在`func Process()`中使用了`func consensus.Finalize()`.
```go
func NewStateProcessor(config *params.ChainConfig, bc *BlockChain, engine consensus.Engine) *StateProcessor {
	return &StateProcessor{
		config: config,
		bc:     bc,
		engine: engine,
	}
}
```

`processor`在`func (bc *BlockChain) InsertChain(chain types.Blocks) (int, error)`处有调用(BlockChain内部只有这一处调用),
```go
receipts, logs, usedGas, err := bc.processor.Process(block, state, bc.vmConfig)
```
用来根据待插入的块来更新状态.


## protocolManager

`protocolManager`仅使用`engine`生成`validator`, 使用了接口的`func VerifyHeader()`. 
```go
	validator := func(header *types.Header) error {
		return engine.VerifyHeader(blockchain, header, true)
	}
```
validator进一步生成`fetcher`, `fetcher`是`protocolManager`的成员变量.
```go
manager.fetcher = fetcher.New(blockchain.GetBlockByHash, validator, manager.BroadcastBlock, heighter, inserter, manager.removePeer)
```
`fetcher`是用来从获取块的
> Fetcher is responsible for accumulating block announcements from various peers and scheduling them for retrieval.

使用engine来验证头部.


## miner
`miner`内部对engine判断, 若为PoW则可以获取HashRate, 该接口也是在`consensus.go`里定义. `miner`用`engine`生成`worker`. 除此`miner`本身对`engine`没有别的使用.
```go
worker:   newWorker(config, engine, common.Address{}, eth, mux)
```

在`worker`中, 有两处使用`engine`, 分别为
```go
func (self *worker) commitNewWork() {
	...
	if err := self.engine.Prepare(self.chain, header); err != nil {
		log.Error("Failed to prepare header for mining", "err", err)
		return
	}
	...
	if work.Block, err = self.engine.Finalize(self.chain, header, work.state, work.txs, uncles, work.receipts); err != nil {
		log.Error("Failed to finalize block for sealing", "err", err)
		return
	}
	...
```
分别来准备新块的头部和验证块.