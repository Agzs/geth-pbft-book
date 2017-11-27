### 从handle(peer)倒推，寻找被调用关系

在`p2p.Server.run()`函数，无限`for`循环中，执行`select`中`case c := <-srv.addpeer:`分支的操作，进行`handshake`，并且`goroutine`启动`Server.runPeer()`，并进一步调用`peer.run()`，最终在`startProtocols()`中调用`proto.Run(p,rw)`，该Run()函数为`handler.go`中`NewProtocolManager()`中的函数类型变量`Run`，该变量最终调用`handle(peer)`，最终进行peer的其他操作。
```go
node.Start() >> server.Start() >> Server.run() >> go, Server.runPeer() >> peer.run() >> startProtocols() >> proto.Run(p,rw)
```
`p2p`中的`Server`的成员变量`addpeer`为`conn`类型的`channel`，在`Server.checkpoint(c,srv.addpeer)`中的`select`选择执行`srv.addpeer <- c`，而`checkpoint()`函数仅在`Server.SetupConn()`中被调用，而`SetupConn()`在`Server.listenLoop()`中被调用，`listenLoop()`函数中包含我们之前进行`admin.addPeer()`所用到的`enode`信息，由`server.makeSelf()`函数生成
```go
server.Start() >> startListening() >> go，listenLoop() >> SetupConn() >> checkpoint()，`srv.addpeer <- c`
```
`server.Start()`函数，首先初始化`server`的成员变量，包含`addpeer`、`addstatic`等各种`channel`；然后根据`server`继承`config`中的各种`XXDiscovery`进行各自的操作。

==========================================================================
### 从admin.addPeer()正推，找调用关系

命令行运行`admin.addPeer()`实际调用`node/api.go`中的`AddPeer(url)`，该函数根据`url`初始化一个`p2p/discover.Node`，然后调用`p2p.server.AddPeer(node)`，最终返回`true`

`p2p.server.AddPeer(node)`仅包含一个`select`语句，将`node`发送到`srv.addstatic`，该`channel`是`p2p.Server`的一个`discover.Node`类型的成员变量，定义如下：
```go
// Node represents a host on the network. The fields of Node may not be modified.
type Node struct {
	IP       net.IP // len 4 for IPv4 or 16 for IPv6
	UDP, TCP uint16 // port numbers
	ID       NodeID // the node's public key

	// This is a cached copy of sha3(ID) which is used for node distance calculations.
        // This is part of Node in order to make it possible to write tests that need a node at a certain distance.
	// In those tests, the content of sha will not actually correspond with ID.
	sha common.Hash

	// whether this node is currently being pinged in order to replace it in a bucket
	contested bool
}
```

在`p2p.Server.Run()`函数，无限`for`循环中，执行`select`中`case n := <-srv.addstatic:`分支的操作，调用`dialstate.addStatic(n)`，该方法是`dialer`接口的一个方法，由`p2p/dial.go`中的`dialstate`实现，该函数仅执行`s.static[n.ID] = &dialTask{flags: staticDialedConn, dest: n}`，`dialTask`的结构体如下：
```go
// A dialTask is generated for each node that is dialed. Its fields cannot be accessed while the task is running.
type dialTask struct {
	flags        connFlag
	dest         *discover.Node
	lastResolved time.Time
	resolveDelay time.Duration
}
```

`s.static`在同一文件中的`newTasks()`中被使用，该函数中通过`for`循环遍历`s.static`，然后针对每一个`static`，调用`s.checkDial(t.dest, peers)`进行连接检查，若没有错误产生，则会将`static`添加到`newtasks`中，另外，`newTasks()`函数的返回值为`newtasks`.

`newTasks()`函数在`Server.run()`中在函数类型变量`scheduleTasks`中被调用，`newTasks()`的返回值`newtasks`随后作为`startTasks()`的参数，`startTasks()`会根据每个`task`的类型，`goroutine`启动`Do(server)`函数；而`scheduleTasks()`在随后的无限`for`循环中被调用。
```go
Server.run() >> for, scheduleTasks() >> newTasks() ==> startTasks() >> task.Do()
```
由于`admin.addPeer`生成的`task`为`dialTask`，所以在`startTasks()`中调用的`Do()`函数如下：
```go
func (t *dialTask) Do(srv *Server) {
	if t.dest.Incomplete() { //=>IP为nil
		if !t.resolve(srv) {
			return
		}
	}
	success := t.dial(srv, t.dest) //=>尝试实际连接
	// Try resolving the ID of static nodes if dialing failed.
	if !success && t.flags&staticDialedConn != 0 {
		if t.resolve(srv) {
			t.dial(srv, t.dest)
		}
	}
}
```
该函数调用`dial()`尝试实际连接，`dial()`函数如下：
```go
func (t *dialTask) dial(srv *Server, dest *discover.Node) bool {
	fd, err := srv.Dialer.Dial(dest) //=>fd为net.Conn类型
	if err != nil {
		log.Trace("Dial error", "task", t, "err", err)
		return false
	}
	mfd := newMeteredConn(fd, false)
	srv.SetupConn(mfd, t.flags, dest)
	return true
}
```
`dial()`函数调用`srv.Dialer.Dial(dest)`，该方法由`func (t TCPDialer) Dial(dest *discover.Node) (net.Conn, error){}`实现，建立一个TCP连接，其后会调用go语言库函数，不再深究，如下：
```go
// Dial creates a TCP connection to the node
func (t TCPDialer) Dial(dest *discover.Node) (net.Conn, error) {
	addr := &net.TCPAddr{IP: dest.IP, Port: int(dest.TCP)}
	return t.Dialer.Dial("tcp", addr.String()) //=> 该函数位于/usr/lib/go/src/net/dial.go，为go语言库函数
}
```
随后，`dial()`函数调用`srv.SetupConn(mfd, t.flags, dest)`，如下：
> SetupConn runs the handshakes and attempts to add the connection as a peer. It returns when the connection has been added as a peer or the handshakes have failed.
```go
func (srv *Server) SetupConn(fd net.Conn, flags connFlag, dialDest *discover.Node) {
	// Prevent leftover pending conns from entering the handshake.
	...
        初始化handshake连接c，类型为conn

	// Run the encryption handshake.
        ...
        调用rlpx.go中的doEncHandshake(srv.PrivateKey, dialDest)
	
	// For dialed connections, check that the remote public key matches.
	...
        判断是否匹配

        ...
        调用srv.checkpoint(c, srv.posthandshake)，与“从handle(peer)倒推，寻找被调用关系”中的checkpoint()为同一个函数，
        但是参数不同，srv.posthandshake为channel，表示conn已通过encHandShake(身份已知，但是尚未验证）

	// Run the protocol handshake
	...
        调用c.doProtoHandshake(srv.ourHandshake)，该函数会通过p2p.Send()，发送handshakeMsg标识的消息；
        然后，调用readProtocolHandshake()读取接收到的handshakeMsg标识消息。(该标识和pbftMessage标识类似）
	
	c.caps, c.name = phs.Caps, phs.Name
	...
        调用srv.checkpoint(c, srv.addpeer)，与“从handle(peer)倒推，寻找被调用关系”中的checkpoint()为同一个函数，
        参数也完全相同，其后操作见上文。

	// If the checks completed successfully, runPeer has now been launched by run.
}
```

