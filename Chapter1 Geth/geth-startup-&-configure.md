# 摘要

geth指令启动的入口定义在cmd/geth. 通过init函数初始化了启动的函数入口以及启动前和结束后的动作. 程序的实际入口为`cmd/geth/main.go`中定义的`func geth()`, 程序首先初始化`node`, 然后启动, 之后注册`EthService`, 注册过程生成了一个`eth`, `eth`会生成一个`engine`, 在生成`engine`的过程中会选择共识机制, 并调用共识机制的生成函数. 以后的其他关于共识机制的对象皆在`engine`的基础上生成.

# 正文

`cmd/geth`是一个可执行pkg(即包含main函数的pkg), 编译生成名为`geth`可执行文件. 使用`go install -v ./...`或`go install -v ./cmd/geth`编译, 该文件会出现在`$GOPATH/bin`下. 使用`make`编译不会, geth-pbft不使用`make`.

## `cmd/geth/main.go`

### `func init()`

该函数在包初始化时调用, 先于`func main()`执行.

`main.go`包含全局变量`app`, 使用`func utils.NewApp()`初始化. `func utils.NewApp()`定义在`cmd/utils/flags.go`中, 函数内部不包含特殊的配置信息.

`app`是使用`"gopkg.in/urfave/cli.v1"`管理的, 其内部机制尚未深究.

在`init()`定义了`app.Flags`, `app.Before`, `app.After`. 其中, Flags包含了geth包含的各种子命令, 在`./*cmd.go`中定义, 每个command包含了`Action`, `Description`等信息. 例如`chaincmd.go`中定义的`initCommand`, 描述了初始化chain命令的信息.
```go
	initCommand = cli.Command{
		Action:    utils.MigrateFlags(initGenesis),
		Name:      "init",
		Usage:     "Bootstrap and initialize a new genesis block",
		ArgsUsage: "<genesisPath>",
		Flags: []cli.Flag{
			utils.DataDirFlag,
			utils.LightModeFlag,
		},
		Category: "BLOCKCHAIN COMMANDS",
		Description: `
The init command initializes a new genesis block and definition for the network.
This is a destructive action and changes the network in which you will be
participating.

It expects the genesis file as argument.`,
	}
```

`app.Before`应该是定义执行`app.Action`主体之前的动作, 包含了CPU信息, 调用了`utils.SetupNetwork(ctx)`.

`app.After`定义了程序结束后的后续处理动作.
```go
	app.After = func(ctx *cli.Context) error {
		debug.Exit()
		console.Stdin.Close() // Resets terminal mode.
		return nil
	}
```


在`func init()`中, 执行了`app.Action = geth`, 将`app.Action`定义为`func geth()`.

### `func geth(ctx *cli.Context) error`

`cmd/geth/main.go`中定义了`func geth()`. 如下:
```go
func geth(ctx *cli.Context) error {
	node := makeFullNode(ctx)
	startNode(ctx, node)
	node.Wait()
	return nil
}
```

在`func main()`中, 执行了`app.Run(os.Args)`. 即, 在命令行中键入`geth`后, 经过几次跳转之后会运行`func geth()`. 主要的配置信息从`func geth()`开始. 我们将`func geth()`视作事实上的程序入口.

`func geth()`通过`func makeFullNode(ctx *cli.Context) *node.Node`获得`node`实例, 该函数在`config.go`中定义. 其后通过`func startNode(ctx *cli.Context, stack *node.Node)`启动.

### `func startNode(ctx *cli.Context, stack *node.Node)`

> startNode boots up the system node and all registered protocols, after which it unlocks any requested accounts, and starts the RPC/IPC interfaces and the miner.

首先调用了`cmd/utils/cmd.go`中定义的`func StartNode(stack *node.Node)`, 该函数只是注册中断信号等. 其后, 该函数配置了account, wallet, rpc(Remote Procedure Call)等信息, 暂不关心. 最后根据配置信息判断是否开始挖矿, 配置GasPrice, 并启动相关线程.

## `cmd/geth/config.go`

### `func makeConfigNode(ctx *cli.Context) (*node.Node, gethConfig)`
该函数内初始化`eth`, `shh`, `node`的初始配置.
```go
...
	cfg := gethConfig{
		Eth:  eth.DefaultConfig,
		Shh:  whisper.DefaultConfig,
		Node: defaultNodeConfig(),
	}
...
```
其内设置了`EthStatsURL`, 作用尚未清楚.

### `func makeFullNode(ctx *cli.Context) *node.Node`

如上述, 该函数会被`cmd/geth/main.go`中的`func geth()`调用.

```go
func makeFullNode(ctx *cli.Context) *node.Node {
	stack, cfg := makeConfigNode(ctx)

	utils.RegisterEthService(stack, &cfg.Eth)
...
```

首先通过调用了`func makeConfigNode(ctx *cli.Context) (*node.Node, gethConfig)`, 进行了配置, 获得节点实例. 其后执行`utils.RegisterEthService(stack, &cfg.Eth)`, 开始具体eth的配置. 其内便包含了共识机制的设置, 在`cmd/utils/flags.go`中进行了实现.

## `cmd/utils/flags.go`

### `func RegisterEthService(stack *node.Node, cfg *eth.Config)`

该函数根据`SyncMode`是否为`downloader.LightSync`判断使用light client或std client. 若使用标准客户端, 会进一步执行`fullNode, err := eth.New(ctx, cfg)`来生成节点, 共识机制的配置主要从`eth.New()`开始, 该函数在`eth/backend.go`中定义.

## `eth/backend.go`
```go
func New(ctx *node.ServiceContext, config *Config) (*Ethereum, error) {
	...
	chainDb, err := CreateDB(ctx, config, "chaindata")
	...
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
	...
	vmConfig := vm.Config{EnablePreimageRecording: config.EnablePreimageRecording}
	eth.blockchain, err = core.NewBlockChain(chainDb, eth.chainConfig, eth.engine, eth.eventMux, vmConfig)
	...
	eth.txPool := core.NewTxPool(config.TxPool, eth.chainConfig, eth.EventMux(), eth.blockchain.State, eth.blockchain.GasLimit)
	maxPeers := config.MaxPeers
	...
	eth.protocolManager, err = NewProtocolManager(eth.chainConfig, config.SyncMode, config.NetworkId, maxPeers, eth.eventMux, eth.txPool, eth.engine, eth.blockchain, chainDb);
	eth.miner = miner.New(eth, eth.chainConfig, eth.EventMux(), eth.engine)
	...
	return eth, nil
}
```
内部对协议各个细节进行了定义, 关于共识机制的尤其需要关注`eth.engine`, 该值由`CreateConsensusEngine(ctx, config, chainConfig, chainDb)`定义.

### `CreateConsensusEngine(ctx, config, chainConfig, chainDb)`
```go
func CreateConsensusEngine(ctx *node.ServiceContext, config *Config, chainConfig *params.ChainConfig, db ethdb.Database) consensus.Engine {
	// If proof-of-authority is requested, set it up
	if chainConfig.Clique != nil {
		return clique.New(chainConfig.Clique, db)
	}
	// Otherwise assume proof-of-work
	switch {
	case config.PowFake:
		log.Warn("Ethash used in fake mode")
		return ethash.NewFaker()
	case config.PowTest:
		log.Warn("Ethash used in test mode")
		return ethash.NewTester()
	case config.PowShared:
		log.Warn("Ethash used in shared mode")
		return ethash.NewShared()
	default:
		engine := ethash.New(ctx.ResolvePath(config.EthashCacheDir), config.EthashCachesInMem, config.EthashCachesOnDisk,
			config.EthashDatasetDir, config.EthashDatasetsInMem, config.EthashDatasetsOnDisk)
		engine.SetThreads(-1) // Disable CPU mining
		return engine
	}
}
```
可以看到, 这里根据`config`内的参数选择共识机制, 并调用相关函数获得实例. `ethash.NewFaker()`等在`consensus/ethash`内部定义, 已经进入了共识机制内部.