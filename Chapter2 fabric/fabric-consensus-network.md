# Network
## `Inquirer`
> Inquirer is used to retrieve info about the validating network

```go
type Inquirer interface {
	GetNetworkInfo() (self *pb.PeerEndpoint, network []*pb.PeerEndpoint, err error)
	GetNetworkHandles() (self *pb.PeerID, network []*pb.PeerID, err error)
}
```
`Inquirer`用来获取网络信息. 会返回整个网络上的节点信息. 第一个函数返回`PeerEndpoint`包含了节点的物理位置和`PeerID`等, 而第二个函数则仅仅返回`PeerID`. 两种结构是定义在`protos/fabric.pb.go`中的. `PeerEndpoint.Address`是由ip和端口组成的字符串.
```go
type PeerID struct {
	Name string `protobuf:"bytes,1,opt,name=name" json:"name,omitempty"`
}
...
type PeerEndpoint struct {
	ID      *PeerID           `protobuf:"bytes,1,opt,name=ID,json=iD" json:"ID,omitempty"`
	Address string            `protobuf:"bytes,2,opt,name=address" json:"address,omitempty"`
	Type    PeerEndpoint_Type `protobuf:"varint,3,opt,name=type,enum=protos.PeerEndpoint_Type" json:"type,omitempty"`
	PkiID   []byte            `protobuf:"bytes,4,opt,name=pkiID,proto3" json:"pkiID,omitempty"`
}
```

### Usage
- 作为`consensus.NetworkStack`接口的一部分. 具体见下文对`NetworkStack`的介绍.
```go
// NetworkStack is used to retrieve network info and send messages
type NetworkStack interface {
	Communicator
	Inquirer
}
```

- 作为`consensus/pbft.communicator`接口的一部分. 该接口定义在`consensus/pbft/broadcast.go`.
```go
type communicator interface {
	consensus.Communicator
	consensus.Inquirer
}
```
可以看到, 事实上`communicator`和`NetworkStack`其实是完全一样的接口, 只不过`communicator`定义在`pbft`内部, 仅限`pbft`内部使用. `communicator`是作为`pbft.broadcaster`的成员变量来使用的, 但事实上, `broadcaster`仅使用了`communicator`的单播函数, 而这一部分是由`consensus.Communicator`定义的(pbft内部根据0~n-1的序号标记外部节点, 所以不需要`Inquirer`). 故而, `consensus.Inquirer`在此处其实并没有用处. 完全可以删除. 所以, 实质上, `Inquirer`也是只作为`NetworkStack`的一部分被用到.

### Implement
从`Inquirer`的引用情况可以看出, `Inquirer`的引用都是通过`NetworkStack`实现功能的. 所以具体实现也是通过`NetworkStack`. 将在下面描述.

### Summary
`Inquirer`虽然有两处引用, 但是在`communicator`中, `Inquirer`部分完全没有被用到, 所以其实它只作为`NetworkStack`的一部分被实际用到了. 该接口其实也是完全可以并到`NetworkStack`中, 单列只是为了区别功能.

## `Communicator`
> Communicator is used to send messages to other validators

```go
type Communicator interface {
	Broadcast(msg *pb.Message, peerType pb.PeerEndpoint_Type) error
	Unicast(msg *pb.Message, receiverHandle *pb.PeerID) error
}
```
顾名思义, 该接口定义了广播和单播的函数用于发送消息. 注意到这里的参数, 这里的`receiverHandle`等都是`peer`层的, 不是`pbft`层使用的节点标记, 因此该接口不会直接被`pbft`内部使用.

### Usage
- 作为`consensus.NetworkStack`接口的一部分. 具体见下文对`NetworkStack`的介绍.
```go
// NetworkStack is used to retrieve network info and send messages
type NetworkStack interface {
	Communicator
	Inquirer
}
```
- 作为`consensus/pbft.communicator`接口的一部分. 该接口定义在`consensus/pbft/broadcast.go`.
```go
type communicator interface {
	consensus.Communicator
	consensus.Inquirer
}
```
虽然此处`communicator`是由两个接口合并定义的, 但是, 正如前文所述, 这里面并没有使用`Inquirer`. 所以可以认为`pbft`内定义的`communicator`就是`consensus.Communicator`. `communicator`作为`broadcaster`的成员变量被使用. 由于`pbft`使用的节点标记就是一个数字, `broadcaster`内通过调用`func getValidatorHandle()`获得网络层的handle, 然后通过`communicator`内的函数发送消息.
(对于fabric来说, 其pbft层的handle和网络层的对应很简单, 只是通过字符串操作来并加上前缀就能把pbft层的数字转化成网络层的handle)

### Implement
同样, 对于Usage#1的实现, 将会在下面`NetworkStack`中进行描述. 对于Usage#2, 可以追溯到是在`obcBatch`初始化是赋给`pbft`的, 可以看到, 实际上是把一个`consensus.Stack`接口的数据当作`consensus.Communicator`中交给`pbft`. `Stack`是在`consensus`中定义的一个包括了`NetworkStack`的接口, 实际实现的数据类型为`helper.Helper`. 这里我们不会也不马上描述`Helper`对接口的实现, 但也等到下面介绍`Stack`再一块介绍. 而是选择在下一部分`NetworkStack`介绍完网络部分的接口.

### Summary
该接口主要负责向节点分发消息, 节点handle是采用的网络层的handle, 该接口单独使用只有一处, 是在pbft内, 经过pbft内节点handle向网络层的handle转化后, 该接口会根据handle向网络层发送消息. 虽然这里只是单独使用`Communicator`, 在程序中却是直接用最高级的接口`Stack`来初始化, `Stack`包括了`NetworkStack`也就当然包括`Communicator`, 因此我们在`NetworkStack`中再讨论其实现.

## `NetworkStack`
> NetworkStack is used to retrieve network info and send messages

```go
type NetworkStack interface {
	Communicator
	Inquirer
}
```
`NetworkStack`只是前面两个接口的综合.

### Usage
`NetworkStack`没有被直接使用的地方, 只是在作为`Stack`的一部分被使用. 所以对于其使用情况, 将会在`Stack`的描述中涉及.

### Implement
由于外部没有对`NetworkStack`接口的单独使用, 也就没有了任何结构体特意去实现`NetworkStack`. 所以我们这里将会讨论`Stack`接口里对于`NetworkStack`部分的实现.

对于`Stack`, 我们可以追溯到它是由`Helper`来实现的. `Helper`是在`Engine`初始化过程中初始化的:
```go
engine.helper = NewHelper(coord)
```
其结构体定义在`fabric/consensus/helper/helper.go`.
```go
// Helper contains the reference to the peer's MessageHandlerCoordinator
type Helper struct {
	consenter    consensus.Consenter
	coordinator  peer.MessageHandlerCoordinator
	secOn        bool
	valid        bool // Whether we believe the state is up to date
	secHelper    crypto.Peer
	curBatch     []*pb.Transaction       // TODO, remove after issue 579
	curBatchErrs []*pb.TransactionResult // TODO, remove after issue 579
	persist.Helper

	executor consensus.Executor
}
```
对于`func GetNetworkInfo()`:
```go
// GetNetworkInfo returns the PeerEndpoints of the current validator and the entire validating network
func (h *Helper) GetNetworkInfo() (self *pb.PeerEndpoint, network []*pb.PeerEndpoint, err error) {
	ep, err := h.coordinator.GetPeerEndpoint()
	if err != nil {
		return self, network, fmt.Errorf("Couldn't retrieve own endpoint: %v", err)
	}
	self = ep

	peersMsg, err := h.coordinator.GetPeers()
	if err != nil {
		return self, network, fmt.Errorf("Couldn't retrieve list of peers: %v", err)
	}
	peers := peersMsg.GetPeers()
	for _, endpoint := range peers {
		if endpoint.Type == pb.PeerEndpoint_VALIDATOR {
			network = append(network, endpoint)
		}
	}
	network = append(network, self)

	return
}
```
可以看到其实是通过`Helper`的成员变量`coordinator`来获得自身的节点信息和网络信息, 解析后返回结果. `func GetNetworkHandles()`是依赖`func GetNetworkInfo()`的, 内部是通过先调用`func GetNetworkInfo()`然后把结果进行处理后返回, 而对于`func Broadcast()`和`func Unicast()`则是直接调用的`coordinator`的对应函数. 因此, `Helper`其实是对`coordinator`的一个封装.

`coordinator`是定义在`core/peer/peer.go`中的`MessageHandlerCoordinator`, 已经属于`peer`层内部, 其内部机制不再深究.

### Summary
`NetworkStack`只是作为`Stack`的一部分被使用, 并没有单独实现`NetworkStack`接口的结构体. 对于`Stack`, 由`Helper`来实现, `Helper`对`NetworkStack`部分的实现是依赖于`core/peer`中定义的`MessageHandlerCoordinator`. 该参数在程序启动时会初始化.