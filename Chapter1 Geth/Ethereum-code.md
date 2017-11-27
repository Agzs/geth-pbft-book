### Ethereum Event驱动机制

- 在backend.go中，eth初始化，其中包括protocolManager, miner, txPool
共同的对象
* Engine
- eventMux: 用于本节点不同进程间event驱动过程，如minedNewBlock,txPre等event，需要用eventMux进行订阅、发送
Miner实例化时，实例化worker，然后miner注册Cpuagent，worker也随即注册agent，agent使用engine负责实际的mining过程
Worker向eventMux订阅ChainHead,ChainSide, TxPre等event
- Miner订阅downloader.StartEvent, DoneEvent, FailedEvent等（负责区块链数据下载同步）
- Miner.start()由eth.Startming()调用，一次调用worker和agent的mining过程，实际调用engine的seal函数，调用前对block进行Finalize().
- ProtocolManager订阅minedBlock, txPre等event
- Event的发布有eventMux.Post()调用deliver()进行，最终只要一个event产生，相应的订阅者就会收到该事件
- 对事件的处理：从events.Chan()中读取事件并进行处理

### Mine过程
- Eth.startMining()调用Miner.start()
- Miner.start调用Worker.commitNewWork，产生新的候选区块，调用engine.Finalize()
- 候选block作为work被push到给worker，实际上发送到agent的workch中
- Agent.update读取workCh中的work，创建一个goroutine mine()开始mining
- Angent.mine()调用engine.Seal()进行挖矿
- 结果发送到agent.resultch，即worker.recv中（worker.wait）
- Post NewMinedBlockEvent事件

### Ethereum 通信机制
- eth中的ProtocolManager负责与peer的通信
- PM中有一个newPeerCh，当新的peer发现的时候，从newPeerCh中获取peer的信息，建立一个新的peer(包涵与该peer的网络连接，可进行通信)，存放在peers对象中
- PM通过handle(peer)处理与peer的通信
- 如果需要PM对peers进行广播，可通过for循环对所有peer发送消息。

### Ethereum 节点启动过程
- P2p/server.go
func (srv *Server) run()
* newPeer when addpeer channel has a new peer
```func newPeer(conn *conn, protocols []Protocol) *Peer```
```protomap := matchProtocols(protocols, conn.caps, conn)```
* matchProtocols creates protoRW
* Connection is sent to addpeer by checkpoint
```func (srv *Server) checkpoint(c *conn, stage chan<- *conn) error ```
* runPeer()-- peer.run()-- peer.startProtocols()
* Connection as rw to start protocols
* Var peers contains all connected peers; peers() and peercount() return related information.
- P2p/peer.go:
```
func (p *Peer) startProtocols(writeStart <-chan struct{}, writeErr chan<- error) 
Call Run:  proto.Run(p, proto)
Proto  running  protoRW  Protocol
ProtoRW contains and implements MsgReadWrite: from wstart, werror, w as MsgReadWriter
```
- ProtocolManager: eth/handler.go
```
Run: func(p *p2p.Peer, rw p2p.MsgReadWriter) error
```
If a new peer is connected  pm.handle(peer)  pm.handleMsg(peer)
handleMsg(peer): read msg from peer.rw and process it.
