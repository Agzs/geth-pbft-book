## Plan outline
1. 使用fabric中的pbft实现代码和ethereal的clique共识算法，共同实现ethereum pbft共识机制；
fabric pbft提供了基本的pbft-core算法，clique提供了签名、验证方法，并设计了带签名的block。
2. 消息传递机制采用channel，在pbft包中增加commChan用于传递消息给ProtocolManager，由ProtocolManager进行广播。

~2. 消息传递采用protobuf，沿用fabric中的message.proto，并进行修改；消息传输过程才用ethereum中的ProtocolManager完成，因此engine中需要有一个成员protocolManager对象pm；~

3. 新的共识engine名为PBFT，其实现Engine的所有接口函数: Finalize, verifyHeader, Seal. 共识的开始从PBFT.Seal函数开始；
+ Seal发起PBFT共识算法中的PrePrePare，等到共识结束后，PBFT包使用finishedChan通知Seal，然后Seal返回共识的结果block。

4. 共识的对象为block，与fabric PBFT不同的地方在于fabric对一批tx进行共识，而我们的目标是对候选的block进行共识。

## Update:
+ 11/10: 同时沿用fabric和ethereum的配置方式，前者使用Viper，后者使用XXXConfig（CliqueConfig或者PBFTConfig），前者从config.yaml文件中读取配置信息，后者从源文件中进行配置

+ 11/10: Fabric中的Timer机制：util/events/events.go
> + Timer通过一个Manager进行管理，它们都由TimerFactory产生
> + Timer超时后产生超时事件，该事件将发送给Manager；Manager设置了事件接收者（SetReceiver），将超时事件发送给对应的Receiver，调用该Receiver的ProcessEvent函数；共识机制Engine即为Receiver，需要实现ProcessEvent函数（以实现Receiver接口）。


## 具体的实现：
1. pbft.go: fabric的pbft.go和ethereum中的clique.go两个文件合并成pbft.go
Main modifications;
> 1. 增加了commChan、finishedChan用于传输消息：前者用于传共识消息给ProtocolManager，后者用于通知Seal共识已经结束
>  ~1. add a protocolManager (pm) to the PBFT consensus engine.~
>  2. remove all execution-related parts from pbft.go. because our design does not require any execution of transactions. Executions are already done before consensus with engine.Finalize()
> 3. PBFT节点ID的问题参考下面第三点，这里使用节点ID实例化pbftcore
> 4. Seal函数中使用了channel机制等待共识结束通知
> 5. TODO: config机制(viper还是TOML？)； timer机制还需弄清楚。

2. 修改了message.proto文件，包含了新的PrePrepare消息，其中含有一个block成员：
> 1. 使用protoc进行编译，得到了message.pb.go，然后修改成pbft_messages.go文件，新的消息类型定义文件为pbft_messages.go, 存放在core/types中；(不使用protobuf进行消息编码，采用RLP）
> ~1. 增加了Block，Header， Transaction等消息类型的定义，为了PrePrepare准备；~

3. cmd/geth/main.go: 增加了PBFTIdFlag，用于在命令行(geth --pbftid n)指定节点的ID，因为PBFT算法对每一个节点进行了编号，从0到N－1。

4. cmd/utils/flags.go: 增加了PBFTIdFlag的定义

5. pbft-core.go：
> 1. 增加了两个channel: commChan用于和ProtocolManager进行通信，其中的数据类型为PbftMessage；finsishedChan为共识结束消息channel，用于PBFT在共识结束后通知Seal，其数据类型为struct{}
> 2. 增加函数sendPrePrepare，Seal使用它发起共识算法，其参数为Block，Block将封装成PbftMessage发送给commChan，由ProtocolManager接收并广播
> 3. recvPrePrepare、recvPrepare中消息的发送均通过commChan
> 4. recvCommit使用finishedChan发送共识结束的通知
> 5. 原来的RequestBatch对应于现在的Block，RequestBatchStore对应现在的BlockStore

6. eth/backend.go: CreateConsensusEngine中选择PBFT作为共识算法

7. eth/config.go: pbftid的类型和默认值在这里定义

8. eth/handler.go: ProtoclManager
> 1. 增加了commChan，接收从PBFT传过来的消息
> 2. 启动grouting处理ConsensusMsg
> 3. 增加了函数processConsensusMsg处理共识消息
> 4. BroadcastMsg进行消息的广播
> 5. eth/peer.go中增加了SendMsg，用于实现BroadcastMsg

9. eth/protocol.go: 定义了ConsensusMsg类型为0x11

10. params/config.go: 定义PBFTConfig，类似于CliqueConfig

## Other
> 1. Reserve snapshot of clique, may be useful in PBFT consensus algorithm.

> 2. PBFT algorithm entry point: PBFT.Seal, which calls sendPrePrepare of pfft-core, 
Bridging Seal() and PBFT algorithm.