Welcome to the ethstudy wiki!

This wiki is for people who are interested in Ethereum and Smart Contracts.

## 1. 环境搭建: Virtual Box and Ubuntu
+ (1) Virtual Box安装增强功能，以支持host到guest的粘贴。
+ (2) Ubuntu 16: Run "Software & Update", for "downloaded from" choose sohu.com etc. as the source repository.
+ (3) install golang.
run the following commands in a terminal window:

```ruby
    sudo apt-get update
    sudo apt-get install git
    git clone https://github.com/ethereum/go-ethereum
    sudo apt-get install golang
    cd go-ethereum
    make
```
+ (4) 下载geth源文件（https://github.com/ethereum/go-ethereum）
```ruby
    unzip go-ethereum-master.zip
    (or git clone https://github.com/ethereum/go-ethereum)
```
进入go-ethereum-master目录，编译go-ethereum:

    cd go-ethereum-master
    make
将build/bin/geth拷贝到/usr/bin目录下:

    sudo cp build/bin/geth /usr/bin

+ (5) 安装智能合约编译器solc
```    
    sudo add-apt-repository ppa:ethereum/ethereum
    sudo apt-get update
    sudo apt-get install solc
    which solc
```
##2. 建立两个节点的私有链
+ (1). 新建一个目录ethereum，用于存放区块链等数据。
在~\ethereum\中新建一个文件genesis.json，作为私有链的创世块的说明文件：
```
{
    "nonce": "0x0000000000000042",
    "mixhash": "0x0000000000000000000000000000000000000000000000000000000000000000",
    "difficulty": "0x2000",
    "alloc": {},
    "coinbase": "0x0000000000000000000000000000000000000000",
    "timestamp": "0x00",
    "parentHash": "0x0000000000000000000000000000000000000000000000000000000000000000",
    "extraData": "Custem Ethereum Genesis Block",
    "gasLimit": "0xffffffff"
}
```
+ (2) 在ethereum目录下启动geth，以genesis.json中的创世块开始创建私有链。
```geth --datadir ./ init genesis.json```

+ (3) 启动p2p节点A控制台：
```geth --datadir ./ --networkid 4321 console```
此命令将建立一个p2p网络，网络id为4321，区块链数据存放于ethereum目录中
 
+ (4) 新建另外一个目录ethereum2，启动另外一个node (node B)，使用同样的genesis.json作为创世块，使用不同的端口，启动控制台
```geth --datadir ~/ethereum2 init gensis.json```
```geth --datadir ~/ethereum2 --networkid 4321 --port 30304 console```

+ (5) 在启动后, 在node B控制台中输入命令，让node B连接第一个node A：
在node A控制台输入admin，记录其enode信息
在node B的控制台中，输入如下命令，其中参数为A的enode id：
```
admin.addPeer("enode://bc3fcc53f883cca5a7a3ed979628790e680d811eab2e991a5c7c4767b9c6625363347dfaf56e636679fce8926c63043ffc34898d2b392675f94a54ad548b13ee@[::]:30303")
```

运行admin.peers，验证其已经与另外一个节点连接。

##3. 运行简单的智能合约
+ (1) 简单智能合约在console中编译一个简单的合约代码
```ruby
source = "contract test { function multiply(uint a) returns(uint d) { return a * 7; } }"
clientContract = eth.compile.solidity(source).test
```
编译返回的结果的JSON格式如下
 
其中，
```
	code：编译后的EVM字节码
	info：编译器返回的metadata
	abiDefination:Application Binary Interface定义。具体接口规则参见这里
	compilerVersion：编译此代码的solidity编译器版本
	developerDoc：针对开发者的Natural Specification Format，类似于Doxygen。具体规则参见这里
	language：合约语言
	languageVersion：合约语言版本
	source：源代码
	userDoc：针对用户的Ethereum的Natural Specification Format，类似于Doxygen。
```

编译器返回的JSON结构反映了合约部署的两种不同的路径。info信息真实的存在于区中心化的云中，作为metadata信息来公开验证Blockchain中合约代码的实现。而code信息通过创建交易的方式部署到区块链中。
+ (2) 创建和部署合约
 在进行此步骤前，确保你有一个解锁的账户并且账户中有余额。（可通过mining获得: miner.start(8); miner.stop();可以创建自己独立的测试网络，即自己的区块链，初始化的时候就可以初始化一些有余额的账户）。参考：Test Networks
 
 现在就可以在区块链中创建一个合约了。创建合约的方式是发送一个交易，交易的目的地址是空地址，数据是前面JSON结构中的code字段。
 创建合约的流程如下:
```ruby
var primaryAddress = eth.accounts[0]
var abi = [{ constant: false, inputs: [{ name: 'a', type: 'uint256' } ]}]
var MyContract = eth.contract(abi)
var contract = MyContract.new({from: primaryAddress, data: clientcontract.code}) 
```

+ (3) 获得账户 （personal.newAccount()产生账户）
 
+ (4) 定义一个abi （abi是个js的数组，否则不成功）
 
+ (5) 创建智能合约
 
+ (6) 发送交易部署合约  
 如果交易被pending，如图说明你的miner没有在挖矿，
 
启动一个矿工
```ruby
miner.setEtherbase(eth.primaryAddress)    //设定开采账户
miner.start(8)
eth.getBlock("pending", true).transactions
```
 这时候发现交易已经在区块中
 
 不过会发现，交易还是pending，这是因为该交易区块没有人协助进行运算验证，这时候只需要再启动一个矿工就行了
```
miner.start(8)
```
参考：Private Testnet
发现区块1部署了交易

合约已经记录在区块链上，address已可用。 

+ (7) 与合约进行交互
 可以通过eth.contract()定义一个合约对象，这个对象包含变数合约接口的abi
```ruby
Multiply7 = eth.contract(clientContract.info.abiDefinition);
var myMultiply7 = Multiply7.at(contract.address);
``` 
 
 到这儿，就可以调用智能合约的函数了。
```
myMultiply7.multiply.call(3)
// 或
myMultiply7.multiply.sendTransaction(3, {from: contract.address})
```

可能的Error:
```
Error 1: Cannot start mining without etherbase address: etherbase address must be explicitly specified
Solution: personal.newAccount() to generate a new address.
```

References:
1. https://news.huobi.com/index/detail/8896
区块链开发（零）如何开始学习以太坊及区块链
