## 摘要
在初始化过程中, `peer`获得了`engine`, 该`engine`会启动一个线程一直从`MessageFan`的`out`channel里读取消息送给`consenter`处理.

`out`的消息是`MessageFan`启动线程从多个`in`(`ins`)里读取的.`peer`在初始化中会从`engine`获得`HandlerFactory`, 然后使用`HandlerFactory`生成`ConsensusHandler`, 生成`ConsensusHandler`的过程会创建一个channel, 加入到`ins`. `peer`会通过`func ConsensusHandler.HandleMessage()`将`stream`的消息送给该channel(即`in`).

即stream >> MessageFan.ins >> MessageFan.out >> consenter

## 过程
fabric用`github.com/spf13/cobra`来管理命令. 经过一系列初始化之后, 程序会调用在`peer/node/start.go`中定义的`func serve(args []string) error`, 其中通过命令
```go
peerServer, err = peer.NewPeerWithEngine(secHelperFunc, helper.GetEngine)
```
来获得`Peer`实例, 这里的第二个参数`helper.GetEngine`便是在`consensus/helper/engine.go`定义的`func GetEngine(coord peer.MessageHandlerCoordinator) (peer.Engine, error)`. 在`func peer.NewPeerWithEngine()`中, 有
```go
peer.engine, err = engFactory(peer)
```
即, peer在初始化时, 将自己作为参数传给`func GetEngine()`, 获得一个`engine`实例. `func GetEngine()`即为fabric和共识机制交互的入口. 定义如下:
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
注意到, 前面部分是对`engine`成员变量的初始化, 后面通过`go`指令启动了一个线程, 该线程会一直保持运行. 在`go func() {...}`内部, `func GetOutChannel()`返回了一个`MessageFan`的成员变量`out`, 类型为channel, `for`循环的作用便是一直从该channel读取消息, 然后通过`func engine.consenter.RecvMsg()`送给`consenter`. 

`MessageFan`另外有个成员变量`ins  []<-chan *Message`, 通过调用`func (fan *MessageFan) AddFaninChannel(channel <-chan *Message)`可以向`ins`增加输入的channel, 每一次调用该函数, `MessageFan`会启动一个线程自动开始持续地将来自新channel的消息写入`out`, 进而再被engine送给`consenter`.

`func AddFaninChannel()`仅在`consensus/helper/handler.go`中定义的`func NewConsensusHandler()`中被调用.
> NewConsensusHandler constructs a new MessageHandler for the plugin. Is instance of peer.HandlerFactory

该函数(初始化handler)的过程中, `hanlder`创建了一个channel, 并保存在自己的成员变量中. 该函数也仅仅有一次引用:
```go
// GetHandlerFactory returns new NewConsensusHandler
func (eng *EngineImpl) GetHandlerFactory() peer.HandlerFactory {
	return NewConsensusHandler
}
```
即, `engine`把它进一步封装成成员函数供`peer`使用. 该函数同样是在`func NewPeerWithEngine()`中被使用的:
```go
peer.handlerFactory = peer.engine.GetHandlerFactory()
```

在`peer`接下来可以使用`func  handleChat()`来从`stream`获取消息, 然后交给`handler`.
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

注意到`handler.HandleMessage(in)`, 该`handler`实际上就是通过调用开头说到的`consensus/helper/handler.go`中定义的`func NewConsensusHandler()`返回的`ConsensusHandler`, 类型为
```go
type ConsensusHandler struct {
	peer.MessageHandler
	consenterChan chan *util.Message
	coordinator   peer.MessageHandlerCoordinator
}
```
`func HandleMessage()`的实现为:
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
这里便会通过`handler.consenterChan`将消息发送给`consenter`. 注意到, `return handler.MessageHandler.HandleMessage(msg)`将`msg`重新传回了`peer`, 交给`FMS(?)`处理, 进一步的机制未知.