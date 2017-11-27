## 通信流程
备注：文中所提到的函数都未带参数或参数不全，与源代码不同，并且挑选了关键函数，只用于说明。">>"代表调用。

geth>>startNode()>>utils.StartNode()>>Node.start()

位于`node/node.go`中的 `func (n *Node) Start() error` 用于创建一个p2p node，并运行它。

```go
func (n *Node) Start() error {
	...
	// Initialize the p2p server. This creates the node key and discovery databases.
	n.serverConfig = n.config.P2P
	...
	running := &p2p.Server{Config: n.serverConfig}
	...

	// Otherwise copy and specialize the P2P configuration
	services := make(map[reflect.Type]Service)
	for _, constructor := range n.serviceFuncs {
		// Create a new context for the particular service
		ctx := &ServiceContext{
			...
		}
		for kind, s := range services { // copy needed for threaded access
			ctx.services[kind] = s
		}
		// Construct and save the service
		service, err := constructor(ctx)
		...
		kind := reflect.TypeOf(service)
		...
		services[kind] = service
	}
	// Gather the protocols and start the freshly assembled P2P server
	for _, service := range services {
		running.Protocols = append(running.Protocols, service.Protocols()...)
	}
	if err := running.Start(); err != nil {
		...
	}
	// Start each of the services
	started := []reflect.Type{}
	for kind, service := range services {
		// Start the next service, stopping all previous upon failure
		if err := service.Start(running); err != nil {
			...
		}
		// Mark the service started for potential cleanup
		started = append(started, kind)
	}
	// Lastly start the configured RPC interfaces
	if err := n.startRPC(services); err != nil {
		...
	}
	// Finish initializing the startup
	n.services = services
	n.server = running
	n.stop = make(chan struct{})

	return nil
}
```

其中，Node.Start()又调用了ethereum/go-ethereum/p2p/server.go中的Server.Start()>>goroutine，Server.run()
```go
func (srv *Server) run(dialstate dialer) {
	...
	for {
		...
		select {
		...
		case c := <-srv.addpeer:
			// At this point the connection is past the protocol handshake.
			// Its capabilities are known and the remote identity is verified.
			glog.V(logger.Detail).Infoln("<-addpeer:", c)
			err := srv.protoHandshakeChecks(peers, c)
			if err != nil {
				glog.V(logger.Detail).Infof("Not adding %v as peer: %v", c, err)
			} else {
				// The handshakes are done and it passed all checks.
				p := newPeer(c, srv.Protocols)
				peers[c.id] = p
				go srv.runPeer(p)
			}
			// The dialer logic relies on the assumption that
			// dial tasks complete after the peer has been added or
			// discarded. Unblock the task last.
			c.cont <- err
		...
		}
	}
	...
}
```
Server.run()会调用newPeer()>>matchProtocols() ，matchProtocols() creates structures for matching named subprotocols.

Server.run()会调用runPeer()>>Peer.run()>>Peer.startProtocols()>>proto.Run(p, proto)

其中Run为结构体Protocol的成员变量，类型为func(peer *Peer, rw MsgReadWriter) error，在eth/handler.go中的NewProtocolManager()中赋值
```go
func NewProtocolManager(...) (*ProtocolManager, error) {
	// Create the protocol manager with the base fields
	manager := &ProtocolManager{
		...
	}
	...
	manager.SubProtocols = make([]p2p.Protocol, 0, len(ProtocolVersions))
	for i, version := range ProtocolVersions {
		...
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
		})
	}
	...
}
```
该函数会间接调用ProtocolManager.handle(peer)>>ProtocolManager.handleMsg()

注意：通信过程涉及两个peer.go文件，一个位于ethereum/go-ethereum/p2p/peer.go，结构体为Peer(大写P)；另一个geth-pbft/eth/peer.go，结构体为peer；