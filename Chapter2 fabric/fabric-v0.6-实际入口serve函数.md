
在[fabric v0.6 startup](https://github.com/Agzs/hyperledger/wiki/fabric-v0.6-startup)中分析到node.Cmd()通过nodeStartCmd调用func serve(args []string) error函数，本节主讲该函数。

## func serve(args []string) error

该函数位于`peer/node/start.go`中，主要分析几个比较重要的函数，如下：

### 1、peer.CacheConfiguration()
位于`core/peer/config.go`中，主要获取并设置和缓存一些经常用到的变量，包含两个比较重要的函数类型变量：`getLocalAddress` 和 `getPeerEndpoint`，并且在函数体后面直接被执行。
`getLocalAddress()`用于获取本地工作的`peer`的套接字(IPaddress:port)
`getPeerEndpoint()`用于获取`Peer`的实例`PeerEndpoint`，其成员变量为`peerID、peerAddress、peerType`。

### 2、peer.GetPeerEndpoint()
位于`core/peer/config.go`中，通过`if`判断`configurationCached`真值而决定是否调用`cacheConfiguration()`，虽然`cacheConfiguration()`在程序中多处被调用，但是其先决条件`configurationCached`的真值决定，在`CacheConfiguration()`函数被调用后，`configurationCached`真值将被设为`true`，所以`cacheConfiguration()`函数将不会在运行，进一步调用`CacheConfiguration()`的操作也不会执行。实际上，`cacheConfiguration()`和`CacheConfiguration()`都只运行一次。注意区分大小写！！！

`GetPeerEndpoint()`最终会获得前期执行`peer.CacheConfiguration()`时设置的全局变量`peerEndpoint`。

### 3、createEventHubServer()


### 4、core.SecurityEnabled()
位于`core/config.go`中，调用执行原理同`GetPeerEndpoint()`，只不过两者调用的同名函数不是同一个。

### 5、db.Start()
位于`core/db/db.go`中，init the openchainDB instance and open the db.

`Start()` >> `openchainDB.open()`

### 6、grpcServer := grpc.NewServer(opts...)
位于`vendor/google.golang.org/grpc/server.go`，新建一个全新的`gRPC server`.

### 7、secHelper, err := getSecHelper()
>this should be called exactly once and the result cached.
>NOTE- this crypto func might rightly belong in a crypto package and universally accessed.

该函数仅执行一次，根据`SecurityEnabled()`区分`validator`和`non-validator`.

* validator
    * 调用`RegisterValidator(enrollID, nil, enrollID, enrollSecret)`，进一步调用`validator.register(name, pwd, enrollID, enrollPWD, nil)`，向`PKI`注册`validator`，这个函数比较复杂，后期详细介绍。？？？？？？？？？？？？？？
    * 调用`InitValidator(enrollID, nil)`，进一步调用`validator.init(name, pwd, nil)`，这个函数比较复杂，后期详细介绍。注意函数中的函数类型参数？？？？？？？？？？？？？？

* non-validator
    * 调用`RegisterPeer(enrollID, nil, enrollID, enrollSecret)`，进一步调用`peer.register(NodePeer, name, pwd, enrollID, enrollPWD, nil)`，向`PKI`注册`peer`，最终调用的函数和上面`RegisterValidator`最终调用的为同一个。后期介绍？？？？
    * 调用`InitPeer(enrollID, nil)`，进一步调用`peer.init(NodePeer, name, pwd, nil)`，最终调用的函数和上面`InitValidator`最终调用的为同一个。后期介绍？？？？

### 8、registerChaincodeSupport(chaincode.DefaultChain, grpcServer, secHelper)
`secHelper`为`Peer`接口类型的变量，作为`security helper`实体，作用？？？？
* `chaincode.NewChaincodeSupport(chainname, peer.GetPeerEndpoint, userRunsCC, ccStartupTimeout, secHelper)`初始化一个`ChaincodeSupport`实例。作用？？？
* `RegisterChaincodeSupportServer(grpcServer, ccSrv)`注册服务。

### 9、Create the peerServer
```go
	if peer.ValidatorEnabled() {
		logger.Debug("Running as validating peer - making genesis block if needed")
		makeGenesisError := genesis.MakeGenesis()
		if makeGenesisError != nil {
			return makeGenesisError
		}
		logger.Debugf("Running as validating peer - installing consensus %s",
			viper.GetString("peer.validator.consensus"))

		peerServer, err = peer.NewPeerWithEngine(secHelperFunc, helper.GetEngine)
	} else {
		logger.Debug("Running as non-validating peer")
		peerServer, err = peer.NewPeerWithHandler(secHelperFunc, peer.NewPeerHandler)
	}
```
该过程设计两个比较重要的函数：`NewPeerWithEngine()`和`NewPeerWithHandler()`,分析如下：
#### 9.1、NewPeerWithEngine(secHelperFunc, helper.GetEngine)
>NewPeerWithEngine returns a Peer which uses the supplied handler factory function for creating new handlers on new Chat service invocations.
位于`core/peer/peer.go`中，该函数比较复杂，如下：
```go
func NewPeerWithEngine(secHelperFunc func() crypto.Peer, engFactory EngineFactory) (peer *Impl, err error) {
	peer = new(Impl)
	peerNodes := peer.initDiscovery()

	...

	// Initialize the ledger before the engine, as consensus may want to begin interrogating the ledger immediately
	ledgerPtr, err := ledger.GetLedger()
	...

	peer.engine, err = engFactory(peer)
	...
	peer.handlerFactory = peer.engine.GetHandlerFactory()
	...
	peer.chatWithSomePeers(peerNodes)
	return peer, nil

}
```
* 9.1.1、ledger.GetLedger()
> Initialize the ledger before the engine, as consensus may want to begin interrogating the ledger immediately

`GetLedger()` >> `GetNewLedger()` >> `newBlockchain()` 作用？？？

* 9.1.2、engFactory(peer) ==> GetEngine(peer), peer初始化时, 将自身作为参数传给`func GetEngine()`, 获得一个`engine`实例.

```go
func GetEngine(coord peer.MessageHandlerCoordinator) (peer.Engine, error) {
	var err error
	engineOnce.Do(func() {
		engine = new(EngineImpl)
		engine.helper = NewHelper(coord)
		engine.consenter = controller.NewConsenter(engine.helper)
		engine.helper.setConsenter(engine.consenter)
		engine.peerEndpoint, err = coord.GetPeerEndpoint()
		engine.consensusFan = util.NewMessageFan()

		go func() {
			logger.Debug("Starting up message thread for consenter")

			// The channel never closes, so this should never break
			for msg := range engine.consensusFan.GetOutChannel() {
				engine.consenter.RecvMsg(msg.Msg, msg.Sender)
			}
		}()
	})
	return engine, err
}
```

该函数先对`engine`成员变量的初始化, 后面通过启动了一个goroutine, 在`go func() {...}`内部,`for`循环获取并处理`msg`，获取是通过`engine.consensusFan.GetOutChannel()`返回了`MessageFan`的一个类型为`channel`的成员变量`out`, 处理是通过`func engine.consenter.RecvMsg()`送给`consenter`. 

此外，`MessageFan`还有个成员变量`ins  []<-chan *Message`, 通过调用`func (fan *MessageFan) AddFaninChannel(channel <-chan *Message)`可以向`ins`添加`channel`, 每次调用该函数, `MessageFan`都会启动一个goroutine持续地将新`channel`的消息写入`out`, 进而再被`engine`送给`consenter`.

`func AddFaninChannel()`仅在`consensus/helper/handler.go`中定义的`func NewConsensusHandler()`中被调用:
> NewConsensusHandler constructs a new MessageHandler for the plugin. Is instance of peer.HandlerFactory

```go
func NewConsensusHandler(coord peer.MessageHandlerCoordinator, stream peer.ChatStream, initiatedStream bool) (peer.MessageHandler, error) {

	peerHandler, err := peer.NewPeerHandler(coord, stream, initiatedStream)
	if err != nil {
		return nil, fmt.Errorf("Error creating PeerHandler: %s", err)
	}

	handler := &ConsensusHandler{
		MessageHandler: peerHandler,
		coordinator:    coord,
	}

	consensusQueueSize := viper.GetInt("peer.validator.consensus.buffersize")

	if consensusQueueSize <= 0 {
		logger.Errorf("peer.validator.consensus.buffersize is set to %d, but this must be a positive integer, defaulting to %d", consensusQueueSize, DefaultConsensusQueueSize)
		consensusQueueSize = DefaultConsensusQueueSize
	}

	handler.consenterChan = make(chan *util.Message, consensusQueueSize)
	getEngineImpl().consensusFan.AddFaninChannel(handler.consenterChan)

	return handler, nil
}

```

该函数用于初始化handler, `hanlder`创建了一个channel, 并保存在自己的成员变量`consenterChan`中. 该函数也仅仅有一次引用:

```go
// GetHandlerFactory returns new NewConsensusHandler
func (eng *EngineImpl) GetHandlerFactory() peer.HandlerFactory {
	return NewConsensusHandler
}
```

即, `engine`把它进一步封装成成员函数供`peer`使用. 该函数同样是在`func NewPeerWithEngine()`中被调用的:
`peer.handlerFactory = peer.engine.GetHandlerFactory()`

*  9.1.3、peer.engine.GetHandlerFactory()
上文提到的是逆序引用，转换为顺序为：
`peer.engine.GetHandlerFactory()` ==> `EngineImpl.GetHandlerFactory()` >> `NewConsensusHandler` ==> `NewConsensusHandler()` >> `MessageFan.AddFaninChannel()`


* 9.1.4、peer.chatWithSomePeers(peerNodes)
`Impl.chatWithSomePeers()` >> `go Impl.chatWithPeer()` >> `Impl.handleChat(ctx, stream, true)` 从`stream`获取消息, 然后交给`handler`.

```go
// Chat implementation of the the Chat bidi streaming RPC function
func (p *Impl) handleChat(ctx context.Context, stream ChatStream, initiatedStream bool) error {
	deadline, ok := ctx.Deadline()
	peerLogger.Debugf("Current context deadline = %s, ok = %v", deadline, ok)
	handler, err := p.handlerFactory(p, stream, initiatedStream)
	if err != nil {
		return fmt.Errorf("Error creating handler during handleChat initiation: %s", err)
	}
	defer handler.Stop()
	for {
		in, err := stream.Recv()
		if err == io.EOF {
			peerLogger.Debug("Received EOF, ending Chat")
			return nil
		}
		if err != nil {
			e := fmt.Errorf("Error during Chat, stopping handler: %s", err)
			peerLogger.Error(e.Error())
			return e
		}
		err = handler.HandleMessage(in)
		if err != nil {
			peerLogger.Errorf("Error handling message: %s", err)
			//return err
		}
	}
}
```

该函数中的`handler.HandleMessage(in)`, 其中的`handler`通过`p.handlerFactory(p, stream, initiatedStream)`==>`NewConsensusHandler()`获得，实际上就是前面提到的`consensus/helper/handler.go`中定义的`func NewConsensusHandler()`返回的`ConsensusHandler`, 类型为：
```go
type ConsensusHandler struct {
	peer.MessageHandler
	consenterChan chan *util.Message
	coordinator   peer.MessageHandlerCoordinator
}
```

`handler.HandleMessage()`的定义如下:
```go
func (handler *ConsensusHandler) HandleMessage(msg *pb.Message) error {
	if msg.Type == pb.Message_CONSENSUS {
		senderPE, _ := handler.To()
		select {
		case handler.consenterChan <- &util.Message{
			Msg:    msg,
			Sender: senderPE.ID,
		}:
			return nil
		default:
			err := fmt.Errorf("Message channel for %v full, rejecting", senderPE.ID)
			logger.Errorf("Failed to queue consensus message because: %v", err)
			return err
		}
	}

	if logger.IsEnabledFor(logging.DEBUG) {
		logger.Debugf("Did not handle message of type %s, passing on to next MessageHandler", msg.Type)
	}
	return handler.MessageHandler.HandleMessage(msg)
}
```
该函数通过`handler.consenterChan`将消息发送给`consenter`. 注意到, `return handler.MessageHandler.HandleMessage(msg)`将`msg`重新传回给`handler.MessageHandler`, `handler.MessageHandler`在`NewConsensusHandler()`函数中被`peerHandler`赋值，`peerHandler`由`NewPeerHandler()`初始化，实际上真正调用的函数为`Handler.HandleMessage()`，该函数位于`core/peer/handler.go`中，最终调用`handler.FSM.Event(msg.Type.String(), msg)`，即`msg`最终交给`FSM`处理.
>FSM is the state machine that holds the current state. 第三方扩展包，位于vendor/github.com/looplab/fsm/fsm.go中。

#### 9.2、NewPeerWithHandler(secHelperFunc, peer.NewPeerHandler)
>NewPeerWithHandler returns a Peer which uses the supplied handler factory function for creating new handlers on new Chat service invocations.

位于`core/peer/peer.go`中，该函数部分功能和`NewPeerWithEngine()`相同，如下：
```go
func NewPeerWithHandler(secHelperFunc func() crypto.Peer, handlerFact HandlerFactory) (*Impl, error) {
	...
	peer.handlerFactory = handlerFact
	...
	ledgerPtr, err := ledger.GetLedger()
	...
	peer.chatWithSomePeers(peerNodes)
	return peer, nil
}
```

* 9.2.1、peer.handlerFactory = handlerFact
`handlerFact()` ==> `peer.NewPeerHandler()`，该函数在上文中有涉及,该函数主要返回一个全新的Peer Handler，可能会调用handler.initiatedStream()初始化Stream.

* 9.2.2、ledger.GetLedger() 同上文提到的。

* 9.2.3、peer.chatWithSomePeers(peerNodes) 同上文提到的，最终都是调用`Handler.HandleMessage()`

### 10、注册服务器的函数
```go
        // Register the Peer server
	pb.RegisterPeerServer(grpcServer, peerServer)

	// Register the Admin server
	pb.RegisterAdminServer(grpcServer, core.NewAdminServer())

	// Register Devops server
	serverDevops := core.NewDevopsServer(peerServer)
	pb.RegisterDevopsServer(grpcServer, serverDevops)

	// Register the ServerOpenchain server
	serverOpenchain, err := rest.NewOpenchainServerWithPeerInfo(peerServer)
	if err != nil {
		err = fmt.Errorf("Error creating OpenchainServer: %s", err)
		return err
	}

	pb.RegisterOpenchainServer(grpcServer, serverOpenchain)
```

### 11、goroutine启动服务器
- grpcServer.Serve(lis) // Start the grpc server. 
- ehubGrpcServer.Serve(ehubLis) // Start the event hub server

### 12、return <-serve
一直阻塞，直到grpc启动遇到错误。如果grpc启动未遇到错误，将永久阻塞。
