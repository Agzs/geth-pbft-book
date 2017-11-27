## 摘要
geth定义了两个`peer`类型, 一个在`p2p`, 另一个在`eth`. geth启动后, 由`eth.ProtocalManager`和`p2p.Server`分别监听`addPeer`的消息, 当收到消息后会创建一个新的`peer`, 并开始持续从`peer`的IO中读取消息并处理.

对于`p2p.Server`, 会直接从网络层获得消息, 它会将关于`p2p`网络的消息进行处理(比如`ping`), 其余消息全部会交给`eth.ProtocalManager`. `eth.ProtocalManager`在收到来自`p2p`的消息后, 会识别消息类型(比如交易, 块请求), 并分别进行处理.

我们将会主要关心`eth.ProtocalManager`对消息的处理, `p2p`层不具体描述.

消息大致传送路径为: 网络层 >> p2p >> eth.

## 过程

在geth启动后, 会通过`eth/backend.go`中定义的`func New(ctx *node.ServiceContext, config *Config) (*Ethereum, error)`生成`Ethereum`实例. 在该函数中, 同时通过`eth.protocolManager, err = NewProtocolManager(...)`生成了一个`ProtocalManager`. 

该类型定义在`eth/handler.go`, 通过`func NewProtocolManager()`初始化. 初始化过程中会注册一个函数:
```go
manager.SubProtocols = append(manager.SubProtocols, p2p.Protocol{
	...
	Run: func(p *p2p.Peer, rw p2p.MsgReadWriter) error {
		peer := manager.newPeer(int(version), p, rw)
		select {
		case manager.newPeerCh <- peer:
			manager.wg.Add(1)
			defer manager.wg.Done()
			return manager.handle(peer)
		case <-manager.quitSync:
			return p2p.DiscQuitting
		}
	},
...
```
该函数会在P2P协议启动时, 开启单独的线程并执行(定义在`p2p/peer.go`). 注意到, 该函数内部会定义: 当`manager.newPeerCh`收到一个新的`peer`后, 会调用`manager.handle(peer)`, 即每一个新的`peer`都会被调用`manager.handle(peer)`.

`func handle()`定义如下:
```go
func (pm *ProtocolManager) handle(p *peer) error {
	...
	for {
		if err := pm.handleMsg(p); err != nil {
			p.Log().Debug("Ethereum message handling failed", "err", err)
			return err
		}
	}
}
```
函数末尾会有一个死循环, 一直重复调用`pm.handleMsg(p)`. `func handleMsg()`就是程序处理消息的地方.
> handleMsg is invoked whenever an inbound message is received from a remote peer. The remote connection is torn down upon returning any error.

**在geth还有一个注册peer并持续调用handleMsg的地方, 在`p2p.Server`. 两者的`peer`类型只是名字一样, 是两个不同的结构. 其也会监听`addPeer`消息, 创建`peer`, 启动`peer`并通过`handleMsg`接收消息(也叫`handleMsg`, 但不是同一个函数), 但里面只会处理关于`p2p`的消息(例如ping), 其他消息作为`subprotocol message`仍会交给`ProtocalManager`处理.**

```go
func (pm *ProtocolManager) handleMsg(p *peer) error {
	// Read the next message from the remote peer, and ensure it's fully consumed
	msg, err := p.rw.ReadMsg()
	...
	// Handle the message depending on its contents
	switch {
	case msg.Code == StatusMsg:
	...
	// Block header query, collect the requested headers and reply
	case msg.Code == GetBlockHeadersMsg:
	...
	case msg.Code == BlockHeadersMsg:
	...
	case msg.Code == GetBlockBodiesMsg:
	...
	case msg.Code == BlockBodiesMsg:
	...
	case msg.Code == TxMsg:
		...
		// Transactions can be processed, parse all of them and deliver to the pool
		var txs []*types.Transaction
		if err := msg.Decode(&txs); err != nil {
			return errResp(ErrDecode, "msg %v: %v", msg, err)
		}
		for i, tx := range txs {
			// Validate and mark the remote transaction
			if tx == nil {
				return errResp(ErrDecode, "transaction %d is nil", i)
			}
			p.MarkTransaction(tx.Hash())
		}
		pm.txpool.AddBatch(txs)

	default:
		return errResp(ErrInvalidMsgCode, "%v", msg.Code)
	}
	return nil
}
```
可以看到, 这里geth根据消息code的不同, 判断消息类型并执行相应操作. 这里的消息是从`peer.rw.ReadMsg()`中读取的.消息本身则是在`p2p/message.go`里定义的. `rw`的类型为`p2p.MsgReadWriter`, 会根据各个`peer`的链接从网络获取消息. 再底层的机制不再深究.