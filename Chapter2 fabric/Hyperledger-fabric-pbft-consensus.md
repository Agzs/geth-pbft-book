## Summary：
+ 系统启动时运行serve(),调用NewPeerWithEngine启动Validator Peer，具备validate功能的peer；

+ 然后调用chatWithSomePeers,chatWithSomePeers调用chatWithPeer，其又调用handleChat，与该peer进行连接，并产生ConsensusHandler，

+ ConsensusHandler处理与peer的连接stream中的消息，即调用HandleMessage（）。此函数判断消息的类型是否属于CONSENSUS类型，将之发送到consenterChan，该channel可由consenter读取并处理；非CONSENSUS类型的消息有MessageHandler处理。

## 代码分析如下：
### 1. 首先我们对Engine的产生过程进行分析，在core/peer/peer.go中：
```
// NewPeerWithEngine returns a Peer which uses the supplied handler factory function for creating new handlers on new Chat service invocations.
func NewPeerWithEngine(secHelperFunc func() crypto.Peer, engFactory EngineFactory) (peer *Impl, err error) {
	peer = new(Impl)
	peerNodes := peer.initDiscovery()
...

	peer.engine, err = engFactory(peer)
	peer.handlerFactory = peer.engine.GetHandlerFactory()
	peer.chatWithSomePeers(peerNodes)
...
}
```
其中包括engine的产生和handlerFactory函数的赋值。其中调用了GetHandlerFactory的到handlerFactory。

### 2. 接下来我们看Impl结构体，其即为区块链系统节点结构体，或者是验证节点VP，或者是普通的Peer(定义在core/peer/peer.go)
```
// Impl implementation of the Peer service
type Impl struct {
	handlerFactory HandlerFactory
	handlerMap     *handlerMap
	ledgerWrapper  *ledgerWrapper
	secHelper      crypto.Peer
	engine         Engine
	isValidator    bool
	reconnectOnce  sync.Once
	discHelper     discovery.Discovery
	discPersist    bool
}
```
其中handlerFactory为函数，由GetHandlerFactory()赋值，我们看GetHandlerFactory如何得到调用函数

### 3. 由EngineImpl代码可看出，GetHandlerFactory实际上就是调用NewConsensusHandler (在consensus/helper/engine.go中)
```
// GetHandlerFactory returns new NewConsensusHandler
func (eng *EngineImpl) GetHandlerFactory() peer.HandlerFactory {
	return NewConsensusHandler
}
```

### 4.接下来分析NewConsensusHandler（consensus/helper/handler.go）:
```
/ NewConsensusHandler constructs a new MessageHandler for the plugin.
// Is instance of peer.HandlerFactory
func NewConsensusHandler(coord peer.MessageHandlerCoordinator,
	stream peer.ChatStream, initiatedStream bool) (peer.MessageHandler, error) {

	peerHandler, err := peer.NewPeerHandler(coord, stream, initiatedStream)
...
	handler := &ConsensusHandler{
		MessageHandler: peerHandler,
		coordinator:    coord,
	}
...
	handler.consenterChan = make(chan *util.Message, consensusQueueSize)
	getEngineImpl().consensusFan.AddFaninChannel(handler.consenterChan)

	return handler, nil
}
```
从NewConsensusHandler可看出，其实现产生一个PeerHandler，具备P2P通信处理的功能；包涵一个MessageHandlerCoordinator，实际为节点自己，即Impl的实例peer。注意handler的consenterChan添加到consensusFan中，即其消息发送到engine的consensusFan，engine会对消息进行处理。

ConsensusHandler的结构体如下：
```
// ConsensusHandler handles consensus messages.
// It also implements the Stack.
type ConsensusHandler struct {
	peer.MessageHandler
	consenterChan chan *util.Message
	coordinator   peer.MessageHandlerCoordinator
}
```
其继承了MessageHandler,并且包含一个channel和一个coordinator.

由上面的分析课可看出，handlerFactory即为NewConsensusHandler，接下来看它什么时候被调用：

### 5. NewConsensusHandler在Impl.handleChat中被调用：
```
// Chat implementation of the the Chat bidi streaming RPC function
func (p *Impl) handleChat(ctx context.Context, stream ChatStream, initiatedStream bool) error {
	deadline, ok := ctx.Deadline()
	peerLogger.Debugf("Current context deadline = %s, ok = %v", deadline, ok)
	handler, err := p.handlerFactory(p, stream, initiatedStream)
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
上面的代码显示handlerChat调用了handlerFactory，开始处理收到的消息，这里handlerFactory返回一个ConsensusHandler类型的handler，其成员变量coord即为peer；stream开始接收消息，并让handler调用HandleMessage进行处理。
从以上代码可以看到
1） NewConsensusHandler中的coord实际上就是Impl的实例，即peer。
2） stream为的消息由handler进行处理。
3） 对于每一个peer，都会产生一个handler处理其传输过来的消息

以上过程的实际效果是，节点在handleChat过程内，通过handlerFactory产生一个ConsensusHandler，ConsensusHandler内产生一个PeerHandler。然后通过stream读取消息，消息将有HandleMessage进行处理。

### 6. handleChat被两个函数调用，分别是chatwithpeers和chat：
```
func (p *Impl) chatWithPeer(address string) error {
	conn, err := NewPeerClientConnectionWithAddress(address)
	if err != nil {
		peerLogger.Errorf("Error creating connection to peer address %s: %s", address, err)
		return err
	}
	serverClient := pb.NewPeerClient(conn)
	ctx := context.Background()
	stream, err := serverClient.Chat(ctx)
...
	err = p.handleChat(ctx, stream, true)
	stream.CloseSend()
...
	return nil
}
```
以上可看出chatWithPeer是与peer建立连接，stream即为网络连接数据流。chatWithPeer由chatWithSomePeers调用，它又是由NewPeerWithEngine调用。当新的peer被发现的时候，PeersDiscovered被调用，它也会调用chatWithSomePeers().
Impl为每一个新发现的peer调用chatWithSomePeers,其会调用chatWithPeer，随机调用handleChat，其会调用handlerFactory产生一个ConsensusHandler，由此Consensushandler的HandleMessage()处理与peer链接的stream中的消息。



### 7. NewPeerWithEngine为Validator，NewPeerWithHandler为非Validator,在peer/node/node.go文件中的serve()函数中：
```
	// Create the peerServer
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


## ConsensusHandler将消息传递给ConsensusEngine
### 1. ConsensusHandler处理消息的函数为
```
// HandleMessage handles the incoming Fabric messages for the Peer
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
实际上它将消息发送给了consenterChan，而在NewConsensusHandler函数中，该Channel是添加到了ConsensusFan.ins,因此需关注consensusFan.out是被谁读取处理。

MessageHandler即为ConsensusHandler的成员PeerHandler，其在core/peer/handler.go中定义，其功能为各种非Consensus消息提供处理过程。


### 2. 从GetEngine可看出，consenter调用了RecvMsg处理consensusFan.out中的消息，GetEngine实际上极为engFactory().
```
// GetEngine returns initialized peer.Engine
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
RecvMsg实际上由obcBatch中的externalEventReceiver执行，然后发送给其成员Manager(一个事件管理器)的events channel。
该Manager的接收者被设置成obcBatch自己，Manager启动后会开始事件循环，陆续发送channel中的事件给obcBatch，让obcBatch调用ProcessEvent处理事件(consensus/util/events.go)。
```
func newObcBatch(id uint64, config *viper.Viper, stack consensus.Stack) *obcBatch {
	var err error

	op := &obcBatch{
		obcGeneric: obcGeneric{stack: stack},
	}

	op.persistForward.persistor = stack

	logger.Debugf("Replica %d obtaining startup information", id)

	op.manager = events.NewManagerImpl() // TODO, this is hacky, eventually rip it out
	op.manager.SetReceiver(op)
	etf := events.NewTimerFactoryImpl(op.manager)
	op.pbft = newPbftCore(id, config, op, etf)
	op.manager.Start()
...
```

obcBatch的ProcessEvent会进而调用pbftcore的ProcessEvent处理PBFT各种事件，obcBatch只处理与Batch相关的事件。
此外，obcBatch还负责通信和签名机制，如下：

### 3. innerStack interface implemented by obcBatch.

```
type innerStack interface {
	broadcast(msgPayload []byte)
	unicast(msgPayload []byte, receiverID uint64) (err error)
	execute(seqNo uint64, reqBatch *RequestBatch) // This is invoked on a separate thread
	getState() []byte
	getLastSeqNo() (uint64, error)
	skipTo(seqNo uint64, snapshotID []byte, peers []uint64)

	sign(msg []byte) ([]byte, error)
	verify(senderID uint64, signature []byte, message []byte) error

	invalidateState()
	validateState()

	consensus.StatePersistor
}
```

pbftCore 中包含一个成员类型为innerStack的consumer，即为obcBatch对象，该consumer为pbftCore提供网络通信和签名验证功能。

## VP节点命名规则

Fabric采用了viper工具进行系统配置，viper可以读取配置文件，环境参数，以及命令行参数，对系统节点等对象进行配置。
Fabric的主要配置文件为peer/core.yaml,该文件配置了VP节点的ID，consensus算法等参数。

在PBFT算法中，每个节点有一个ValidatorID，该ID来自于core.yaml配置的ID，比如core.yaml中设置的ID为vp0，
则PBFT中该节点的ID为0，以此类推。具体可见pbft.go文件中的下列函数：

```
// Returns the uint64 ID corresponding to a peer handle
func getValidatorID(handle *pb.PeerID) (id uint64, err error) {
	// as requested here: https://github.com/hyperledger/fabric/issues/462#issuecomment-170785410
	if startsWith := strings.HasPrefix(handle.Name, "vp"); startsWith {
		id, err = strconv.ParseUint(handle.Name[2:], 10, 64)
		if err != nil {
			return id, fmt.Errorf("Error extracting ID from \"%s\" handle: %v", handle.Name, err)
		}
		return
	}

	err = fmt.Errorf(`For MVP, set the VP's peer.id to vpX,
		where X is a unique integer between 0 and N-1
		(N being the maximum number of VPs in the network`)
	return
}
```
其中注明了“For MVP, set the VP's peer.id to vpX, where X is a unique integer between 0 and N-1”。


### PBFT Request and Message
+ PBFT中，每一个request中包涵一个tx，使用txToRequest将tx封装成Request；
+ RequestBatch包涵多个Requests, obcBatch中的BatchStore也是多个Request组成；
+ Request发送到primary node，由primary node发起PBFT共识过程。

### ProtocolBuffer
+ PBFT 中使用ProtocolBuffer对发送的消息进行定义，定义在message.proto文件，然后使用proton进行编译，得到message.pb.go文件。

