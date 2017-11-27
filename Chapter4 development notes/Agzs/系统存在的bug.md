### 随记
随想随记

### 1、peer连通问题
两两互联，才能广播消息彼此消息，新加入的signer需要和其他所有signer互联，会不会太麻烦？在signer间共享各自连接的peer？

已实现

初步设想：peer间定期广播自己保存的peer，类似同步区块的过程。

后期实现过程中，参考pbftMessage的传递读取过程，编写了peerMsg的相关代码，但是peerMsg可以正常发送；读取的时候，可以读取peer的个数，但是无法读取出具体的peer值，由于peer结构体的定义中包含过多的interface成员变量，导致无法正确解码，这一错误和之前pbftMessage中的payload问题相似。

### 2、pbftID问题
~不管什么用户，即使没有指定，也会默认为0？ 做两套系统？普通用户版的不存在seal和processEvent中的PBFT操作？~

另做一套系统，普通用户只能广播和接收处理block和tx的消息，无法响应pbft的消息，已实现。

~### 3、普通节点问题
普通节点加入到p2p中，就会接收signer广播的消息并进行响应，做两套系统？
同2~

### 4、交易问题
交易的产生者，必须和主节点两两互联，交易才能被处理，如果发生viewchange，主节点变了，连接也得变？ 和问题1类似

### 5、批量模拟
模拟庞大的用户群，每个都得addPeer()...和问题1类似

### 6、PBFT时间timeout设置
VP个数不同，共识成功的时间不同，控制VP个数，3 < n < X ?

### 7、viewchange测试（学姐负责）
哪些情形触发？ timeout设置? 

### 8、主节点分配编号问题
还未测试编号上限，可能会出错

### 9、安全问题？
通过puppeth配置，生成.json文件，比如命名为pbft.json，然后通过`geth --datadir pbft/node2/data init pbft.json`初始化创世区块，该文件是可编辑的，可随便更改signer和余额。

但是，一旦用户随意更改.json文件后，通过该文件生成的创世区块和VP生成的创世区块不同，虽然可以加入到同一个网络，但是由于创世区块不同，该用户无法import新区块，也无法让自己成为新的signer.

### 10、通过propose添加新的signer
~仿照clique的API，编写pbft的API，但是测试时，始终提示pbft未定义，而clique却可以，~ 

API问题已解决，目前投票选举新的signer采用主节点一票通过的方式，已解决

### 11、signer宕机情况测试
模拟主节点退出，其他signer正常运行时的情况。

