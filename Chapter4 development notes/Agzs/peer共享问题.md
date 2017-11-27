### 11.16 push
本次`push`主要解决了`peer`共享问题，即在主节点中运行`admin.addPeer()`命令连接其他VP时，主节点将会向其他VP广播自身保存的`enode`列表，其他VP会自动执行`addPeer`指令进行连接。举个例子，假设系统共有4个`signer`，其中`signer1`为主节点，`signer2、3、4`为VP，连接网络时，只需在`signer1`的`console`界面中分别对其他三个`signer`的`enode`执行`admin.addPeer()`，其他三个`signer`会对另外两个`signer`的`enode`自动执行`addPeer()`操作，这样各个`signer`之间两两相连。

或者，连接网络时，只需在`signer1`的`console`界面中执行`signer2`的`enode`执行`admin.addPeer()`，在`signer2`的`console`界面中执行`signer3`的`enode`执行`admin.addPeer()`，在`signer3`的`console`界面中执行`signer4`的`enode`执行`admin.addPeer()`，依然可以实现各个`signer`之间两两相连。

**需要注意的是，节点的添加只能是已在p2p网络中的节点，执行admin.addPeer()操作，添加新节点的enode！！！**

大体思路：

使用全局变量数组，保存所有与当前peer相连接的enode信息，当前peer每成功执行一次`AddPeer()`操作，就将新执行的enode消息添加到enode表中，然后触发p2p的广播函数，向其他相连接的peer广播自己新处理的enode;

其他节点收到消息后，解析出新的enode信息，然后执行`AddPeer()`操作，此时又会将新执行的enode消息添加到自己的enode表中，然后触发p2p的广播函数，向其他相连接的peer广播自己新处理的enode

如此循环往复，所有相连接的peer共享和自己相连接的新的enode，最终全网中的各个节点都互相连接，有点类似于路由表的广播，但最终各个节点保存的enode表不同，enode的数目呈递减趋势，即越早加入网络的节点保存的enode个数越多，每个节点仅保存它加入网络之后添加的新的enode。

但是，需要注意的是，节点的添加只能是已在p2p网络中的节点，执行admin.addPeer()操作，添加新节点的enode！！！

**注：本次修改直接实现了addPeer和removePeer操作，下面仅以addPeer为例进行说明，removePeer与其类似。**

**注2：enode其实就是一个url，所以本文中enode和url一样**

### node/api.go
通过对`admin.addPeer()`的解析，发现该命令会调用`node/api.go`中的`func (api *PrivateAdminAPI) AddPeer(url string) (bool, error) {}`，所以在该文件中进行修改enode相关操作。

#### AddPeerUrlArray数组
添加一个全局变量AddPeerUrlArray数组，用于保存和当前peer相连通的所有enode，定义如下:
```go
var AddPeerUrlArray []*string    // maintain a current active url table added peers.
```
该变量在`func (api *PrivateAdminAPI) AddPeer(url string)`函数中，每次成功执行`addPeer(peer_url)`操作后，都会将`peer_url`添加到`AddPeerUrlArray`数组

此外，为该变量添加下列函数：
```go
// findItemInArray finds whether a url is in urlArray or not.
func findItemInArray(urlArray []*string, url string) bool {
	for _, u := range urlArray {
		if *u == url {
			return true
		}
	}
	return false
}
```
`findItemInArray()`函数用于判断url(也就是enode)是否在当前的enode列表中，主要用于`addPeer`或`removePeer`时，`urlArray`是否已保存peer的url，防止重复执行addPeer`或`removePeer`操作。

```go
func deleteItemInArray(urlArray []*string, url string) []*string {
	var list []*string
	for _, u := range urlArray {
		if *u == url {
			continue
		}
		list = append(list, u)
	}
	return list
}
```
`deleteItemInArray()`函数用于在enode列表中删除指定的url，该函数应用情形主要有两种：

（1）当执行`RemovePeer(peer_2_url)`时，需要对`AddPeerUrlArray`保存的enode表执行`deleteItemInArray()`操作，删除`peer_2_url`。一方面，防止`peer_2_url`被广播给其他节点又重新运行；另一方面，可以让`peer_2_url`重复使用，下次仍可以重新添加到网络中。

（2）当执行`AddPeer(peer_1_url)`时，需要对`RemovePeerUrlArray`保存的enode表执行`deleteItemInArray()`操作，删除`peer_1_url`，防止`peer_1_url`被广播给其他节点又被remove。


#### AddPeerComm
定义了一个全局变量的channel，用于触发p2p中的广播函数
```go
var AddPeerComm chan *string = make(chan *string)    // trigger to BroadcastAddPeerMsg()
```
该变量在`func (api *PrivateAdminAPI) AddPeer(url string)`函数中，每次成功执行`addPeer(peer_url)`操作后，会将`peer_url`添加到`AddPeerUrlArray`数组，同时将`url`发送到`AddPeerComm`中，而`AddPeerComm`的遍历在`handler.go`中的`processPeerMsg()`函数中，类似于`processConsensusMsg()`中`pbftMessage`的`commChan`。

#### privateAdminAPI
定义一个全局变量的`privateAdminAPI`指针，用于备份初始化时创建的`api`
```go
var privateAdminAPI *PrivateAdminAPI //=>add --Agzs 11.15
```
该指针在下列函数中使用：
```go
// GetPrivateAdminAPI for getting api to call AddPeer() --Agzs 11.15
func GetPrivateAdminAPI() *PrivateAdminAPI {
	return privateAdminAPI
}

// NewPrivateAdminAPI creates a new API definition for the private admin methods of the node itself.
func NewPrivateAdminAPI(node *Node) *PrivateAdminAPI {
	//=>return &PrivateAdminAPI{node: node} --Agzs 11.15
	//=>change. --Agzs
	privateAdminAPI = &PrivateAdminAPI{node: node}
	return privateAdminAPI
}
```
后期调用`PrivateAdminAPI`的方法`XXX()`时，可以通过`GetPrivateAdminAPI().XXX()`进行调用。

#### OutCallAddPeer()
```go
// OutCallAddPeer will call AddPeer to add peer
func (api *PrivateAdminAPI) OutCallAddPeer(url *string) {
	//if findItemInArray(AddPeerUrlArray, *url) || IsSelfENode(*url) { // is self or existed in AddPeerUrlArray?
	if findItemInArray(AddPeerUrlArray, *url) { // existed in AddPeerUrlArray?
		return
	}
	flag, err := api.AddPeer(*url)
	if !flag {
		log.Warn("Add peers faild", "error", err, "url", url)
	} else {
		// delete new url from RemovePeerUrlArray, if it existed in RemovePeerUrlArray
		// ensure a url can reuse, ex(a url add, then remove, and add now)
		RemovePeerUrlArray = deleteItemInArray(RemovePeerUrlArray, *url)
	}

}
```
该函数是对外开放的，主要用于在`eth/handler.go`中的`handleMsg()`函数中使用，当收到`AddPeerMsg`标识的消息时，解析出新peer的url，然后对新的url执行`OutCallAddPeer()`操作，进一步调用`AddPeer()`操作，进行节点连接。

================下方待整理=================

### eth/handler.go
#### AddPeerMsg分支
在handleMsg()函数中，添加AddPeerMsg相应分支，该分支用于读取p2p广播的AddPeerMsg标识的消息，RomovePeerMsg类似，如下：
```go
          case msg.Code == AddPeerMsg:
		log.Info("handleMsg() ---AddPeerMsg-----------") //=>test. --Agzs
		var requestAddPeerMsg *string
		if err := msg.Decode(&requestAddPeerMsg); err != nil {
			log.Info("Decode exists error!!!") //=>test. --Agzs
			return errResp(ErrDecode, "%v: %v", msg, err)
		}
		//log.Info("recvAddPeerMsg", "AddPeerMsg", *requestAddPeerMsg) //=>test. --Agzs
		addPeerMsgHash := types.Hash(requestAddPeerMsg)
		p.MarkAddPeerMsg(addPeerMsgHash)
		log.Info("handleMsg will call AddPeers") //=>test.--Agzs
		node.GetPrivateAdminAPI().OutCallAddPeer(requestAddPeerMsg)
```
该分支下，先进行消息的读取，然后标记该消息以确保不会再次向该peer广播该消息，最后交由`api.go`中的`OutCallAddPeer()`函数处理
#### processPeerMsg()
该函数通过读取api.go中定义的`AddPeerComm`中的消息，触发广播消息，`RemovePeerComm`等同。
```go
func (self *ProtocolManager) processPeersMsg() {
	/// Get message from commChan, which is sent by PBFT consensus algorithm.
	log.Info("protocolManager process peers msg!") //=>test. --Agzs

	for addPeerMsg := range node.AddPeerComm {
		self.BroadcastAddPeers(addPeerMsg)
	}

	for removePeerMsg := range node.RemovePeerComm {
		self.BroadcastRemovePeers(removePeerMsg)
	}
}
```
该函数在`protocolManager`的`Start()`方法中进行调用，类似于`processConsensusMsg()`的调用
```go
func (pm *ProtocolManager) Start() {
	...
	/////////////////////////////////////
	/// for consensus message processing. --Zhiguo 04/10
	go pm.processConsensusMsg()
	//=> for shared peers --Agzs 11.16
	go pm.processPeersMsg()
	/////////////////////////////////////
	...
}
```
广播函数如下，主要用于将新的`enode`进行广播，类似于`pbftMessage`的广播
```go
// BroadcastAddPeers will propagate a addPeerMsg to all peers which are not known to
// already have the given addPeerMsg.
//=>--Agzs 11.15
func (pm *ProtocolManager) BroadcastAddPeers(addPeerMsg *string) {
	log.Info("pm.BroadcastAddPeers() start------------") //=>test. --Agzs
	//=> add PeerWithoutMsg() start. --Agzs

	hash := types.Hash(addPeerMsg)
	peers := pm.peers.PeersWithoutAddPeerMsg(hash)

	for _, peer := range peers {
		log.Info("peer broadcast addPeerMsg", "peer", peer.id, "send msg's hash:", hash) //=>test. --Agzs
		peer.SendAddPeerMsg(addPeerMsg)
	}
	log.Info("pm.BroadcastAddPeers() end------------") //=>test. --Agzs

	log.Trace("Broadcast addPeersMsg", "hash", hash, "recipients", len(pm.peers.peers)) //=> peers ->  pm.peers.peers --Agzs
}

// BroadcastRemovePeers will propagate a removePeerMsg to all peers which are not known to
// already have the given removePeerMsg.
//=>--Agzs 11.15
func (pm *ProtocolManager) BroadcastRemovePeers(removePeerMsg *string) {
	log.Info("pm.BroadcastRemovePeers() start------------") //=>test. --Agzs
	//=> add PeerWithoutMsg() start. --Agzs

	hash := types.Hash(removePeerMsg)
	peers := pm.peers.PeersWithoutRemovePeerMsg(hash)

	for _, peer := range peers {
		log.Info("peer broadcast removePeerMsg", "peer", peer.id, "send msg's hash:", hash) //=>test. --Agzs
		peer.SendRemovePeerMsg(removePeerMsg)
	}
	log.Info("pm.BroadcastRemovePeers() end------------") //=>test. --Agzs

	log.Trace("Broadcast removePeersMsg", "hash", hash, "recipients", len(pm.peers.peers)) //=> peers ->  pm.peers.peers --Agzs
}
```

### eth/peer.go
在peer.go中的peer结构体中添加下列两个属性，用于标记该peer是否已接收过`AddPeerMsg`和`RemovePeerMsg`，确保不会重复发送消息
```go
//=> add knownXXXPeerMsg to add or remove peer --Agzs 11.15
	knownAddPeerMsg    *set.Set // Set of addPeerMsg hashes known to be known by this peer
	knownRemovePeerMsg *set.Set // Set of removePeerMsg hashes known to be known by this peer
```
此外，针对上述两个属性，实现上述功能，需要添加下列函数：
```go
//=> add MarkAddPeerMsg() for knownAddPeerMsg --Agzs 11.15
// MarkAddPeerMsg marks a addPeerMsg as known for the peer, ensuring that the addPeerMsg will
// never be propagated to this particular peer.
func (p *peer) MarkAddPeerMsg(hash common.Hash) {
	// If we reached the memory allowance, drop a previously known block hash
	for p.knownAddPeerMsg.Size() >= maxKnownAddPeerMsgs {
		p.knownAddPeerMsg.Pop()
	}
	p.knownAddPeerMsg.Add(hash)
}

//=> add for KnownAddPeerMsg. --Agzs
// PeersWithoutAddPeerMsg retrieves a list of peers that do not have a given addPeerMsg
// in their set of known hashes.
func (ps *peerSet) PeersWithoutAddPeerMsg(hash common.Hash) []*peer {
	ps.lock.RLock()
	defer ps.lock.RUnlock()

	list := make([]*peer, 0, len(ps.peers))
	for _, p := range ps.peers {
		if !p.knownAddPeerMsg.Has(hash) {
			list = append(list, p)
		}
	}
	return list
}
```
另外，自定义p2p中的Send函数，发送`AddPeerMsg`标识的消息
```go
func (p *peer) SendAddPeerMsg(addPeerMsg *string) error {
	log.Info("peer.SendAddPeerMsg() start", "url", *addPeerMsg) //=>test. --Agzs
	p.knownAddPeerMsg.Add(types.Hash(addPeerMsg))
	
        return p2p.Send(p.rw, AddPeerMsg, addPeerMsg)
}
```

### eth/protocol.go

```go
//=>var ProtocolLengths = []uint64{17, 8} //=> --Agzs
var ProtocolLengths = []uint64{25, 8} //=> +1, since add ConsensusMsg. --Agzs


AddPeerMsg    = 0x17 //=>for addPeer--Agzs 11.15
RemovePeerMsg = 0x18 //=>for removePeer --Agzs 11.15

```
### ethereum/go-ethereum/p2p/server.go
注：该过程属于补充，并未在项目中涉及，在此做下标记，以防后期使用

定义全局变量`IsSelfNode`，用于保存当前运行结点自身的enode
```go
//=> for self enode. --Agzs 11.20
var IsSelfNode *string = nil

func localAddr() string {
	conn, err := net.Dial("udp", "google.com:80")
	if err != nil {
		log.Warn(err.Error())
		return ""
	}
	defer conn.Close()
	return conn.LocalAddr().String()
}

// listenLoop runs in its own goroutine and accepts
// inbound connections.
func (srv *Server) listenLoop() {
	defer srv.loopWG.Done()
	//=>log.Info("RLPx listener up", "self", srv.makeSelf(srv.listener, srv.ntab))
	//================> --Agzs 11.20
	if IsSelfNode == nil {
		node := srv.makeSelf(srv.listener, srv.ntab)
		Host := localAddr() // Host = ip:prot
		host, _, err := net.SplitHostPort(Host) // host = ip, type "string"
		if err != nil {
			log.Warn("invalid host: %v", err)
		}
		node.IP = net.ParseIP(host)
		str := node.String()
		IsSelfNode = &str
		log.Info("RLPx listener up", "self", node)
	} else {
		log.Info("RLPx listener up", "self", srv.makeSelf(srv.listener, srv.ntab))
	}
	//================> --Agzs
        ...
}
```