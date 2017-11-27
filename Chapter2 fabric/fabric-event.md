fabric中关于事件处理的大概包括: 网络层的消息传递, 从网络层生成时间向`consensus`的传递, `consensus`内部的时间传递, 以及`consensus`向外的事件传递.

对于网络层, 我们不关心其机制.

对于网络层向`consensus`, fabric采用了`MessageFan`, 在程序初始化时会启动, 然后不断地交给`consensus`处理(进入`consensus`内部).

在`consensus`内部, 采用了`event.Manager`, 每一个需要处理事件的结构都需要实现函数`func ProcessEvent()`, 然后初始化一个`event.Manager`, 需要该结构处理的事件都将发送给`event.Manager`, 由`Manager`是一个一直运行的线程, 在收到消息后就会调用`func ProcessEvent()`来处理事件.

对于`consensus`向外的发送, 是指向外部节点发送消息, 是通过一层层的调用, 最终调用到网络层的广播, 单播函数.

- [fabric event (stream to consenter)](https://github.com/yeongchingtarn/geth-pbft/wiki/fabric-event-(stream-to-consenter))
- [fabric event (consenter to pbft)](https://github.com/yeongchingtarn/geth-pbft/wiki/fabric-event-(consenter-to-pbft))
- [fabric event (pbft to network)](https://github.com/yeongchingtarn/geth-pbft/wiki/fabric-event-(pbft-to-network))