## Clique PoA consensus 建立Private chain
### Ethereum Proof of Authority
Proof of Authority 是直接指定某些节点拥有记帐权，其他节点进行验证，授权的节点生产Block，轮流记账。

geth >= 1.6实现了Clique的共识机制，[ethereum/EIPs#225](https://github.com/ethereum/EIPs/issues/225)。

### 安装geth
#### 第一种方法：
由于go-ethereum 使用golang 开发的，下面是通过命令行方式安装的geth1.7，用到了ppa 。
 ![这里写图片描述](http://img.blog.csdn.net/20171005105033031?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvY29kZV9zZWdtZW50/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

查看geth版本号
 ![这里写图片描述](http://img.blog.csdn.net/20171005105142132?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvY29kZV9zZWdtZW50/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

#### 第二种方法：
geth & tools 1.6 下载地址： https://ethereum.github.io/go-ethereum/downloads/

找到对应的OS 下载相应版本，一定要下载 geth & tools 的版本，因为后期会使用 geth 1.6 版本中一个创建 Private chain 的工具 puppeth 来建立 Clique Private chain。

将下载的.tar.gz文件提取出来，解压后的目录下全是可执行文件，将所有的可执行文件移动到/usr/bin目录下，如果之前安装过geth，应在/usr/bin/下执行$ sudo rm geth 然后在解压的目录下执行$ sudo mv XXX /usr/bin。

### 环境准备 
创建4个文件夹，分别命名为node1、node2、signer1、signer2。node是普通节点，用于后期节点间发起交易；在接下来的实验中，signer1设置为创世块指定的授权节点；signer2为后期加入信任名单的授权节点。
 ![这里写图片描述](http://img.blog.csdn.net/20171005105329233?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvY29kZV9zZWdtZW50/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

 ### 建立Ethereum 帐号
为每个节点建立一个Ethereum 账户。指令`geth --datadir ./data account new`是指使用当下目录下的`data`目录当作`geth`存放资料的地方，并且创一个新的Account。在刚刚创建的node1, node2, signer1, signer2 目录下分别运行相同指令来创建一个账户，如下所示。

在node1文件夹下创建账户，密码自拟，可设为123：
 ![这里写图片描述](http://img.blog.csdn.net/20171005105441492?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvY29kZV9zZWdtZW50/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

在node2文件夹下创建账户，密码自拟，可设为123：
 ![这里写图片描述](http://img.blog.csdn.net/20171005105457208?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvY29kZV9zZWdtZW50/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

在signer1文件夹下创建账户，密码自拟，可设为123：
![这里写图片描述](http://img.blog.csdn.net/20171005105511038?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvY29kZV9zZWdtZW50/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

在signer2文件夹下创建账户，密码自拟，可设为123：
 ![这里写图片描述](http://img.blog.csdn.net/20171005105529355?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvY29kZV9zZWdtZW50/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

以下是我创建的每个角色的Account address:
node1:   0x0a9e3b2ba71cd3c6b3281bbb2a4ed60eadcd7581
node2:   0x42859e6a0c3bdbc3bf9078e59eab655826a540ee
signer1:  0x5824cd5f32da04ecbdc877200f5ac91b5a0e1a5e
signer2:  0x287ea6d4254cabedd72d2e7fc6a0d1e6abd06eb7

注：这些地址最好是保存在txt文件中，以备后期使用。

### 设置创世块
Clique 是将授权节点的相关信息放在Block Header中，必须设置创世块才可以让授权机制生效。

Clique是将授权的信息放在extraData中，但信息结构的格式并没有那么直观，所以在此使用geth 1.6提供的建立Private Chain的工具[puppeth](https://translate.googleusercontent.com/translate_c?depth=1&hl=zh-CN&prev=search&rurl=translate.google.com.hk&sl=zh-TW&sp=nmt4&u=https://blog.ethereum.org/2017/04/14/geth-1-6-puppeth-master/&usg=ALkJrhgYZNq-ZBuxjRYx4iFWI-4zQeKQ9w)来建立创世块， puppeth是个互动式的程序，直接启动照着指示输入相关信息。
设置Private chain名称，假定为poa_for_fun，如下：
 ![这里写图片描述](http://img.blog.csdn.net/20171005105717716?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvY29kZV9zZWdtZW50/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

设定新的创世块，这里选2，如下：
 ![这里写图片描述](http://img.blog.csdn.net/20171005105747251?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvY29kZV9zZWdtZW50/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

选择共识机制，选2，Clique PoA，如下：
 ![这里写图片描述](http://img.blog.csdn.net/20171005105802082?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvY29kZV9zZWdtZW50/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

设置生产出一个Block的时间，在这里设10 秒，可自拟，如下：
 ![这里写图片描述](http://img.blog.csdn.net/20171005105813404?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvY29kZV9zZWdtZW50/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

授权节点。这里使用上面的Signer1的address，如下：
 ![这里写图片描述](http://img.blog.csdn.net/20171005105833652?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvY29kZV9zZWdtZW50/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

给账户设置一些ether。这里选node1和signer1的address，可自拟，如下：
 ![这里写图片描述](http://img.blog.csdn.net/20171005105852552?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvY29kZV9zZWdtZW50/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

Network Id，直接用random；添加genesis 的，留空，如下：
 ![这里写图片描述](http://img.blog.csdn.net/20171005105903069?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvY29kZV9zZWdtZW50/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

存档，选2，如下：
 ![这里写图片描述](http://img.blog.csdn.net/20171005105919546?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvY29kZV9zZWdtZW50/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

ctrl+c 退出，查看当前目录会看到一个poa_for_fun.json 文件。
 ![这里写图片描述](http://img.blog.csdn.net/20171005105937251?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvY29kZV9zZWdtZW50/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)
 
### 初始化私有链

使用`geth init` 指令，分别初始化4 个node 的datadir，比如初始化node1可执行：`geth --datadir node1/data init poa_for_fun.json`，如下：
![这里写图片描述](http://img.blog.csdn.net/20171005110036279?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvY29kZV9zZWdtZW50/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)
![这里写图片描述](http://img.blog.csdn.net/20171005110049823?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvY29kZV9zZWdtZW50/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

### 启动client，设置peers通信
分别在node1, node2目录下，运行如下指令，启动geth：
```
 geth --datadir ./data --networkid 55661 --port 2000 console 
```

这里的参数需要特别注意：
* ① datadir 先前的步骤已经在每个节点各自的目录下都建立了data 目录。
* ② networkid geths之间一定都要用同一个值才可以互相通信，比如实验中的55661。
* ③ port geths之间通信时，监听的一个port，由于四个节点都在本机，所以这里必须都指定不同的值，使用node1对应2000, node2对应2001, signer1对应2002, signer2对应2003。
 ![这里写图片描述](http://img.blog.csdn.net/20171005110248095?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvY29kZV9zZWdtZW50/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)
 ![这里写图片描述](http://img.blog.csdn.net/20171005110301987?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvY29kZV9zZWdtZW50/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)
 
如果节点是授权打包block的节点，启动时要先unlock account，这样才可以打包交易。启动后会要求输入当时创建account时的passphrase。比如，启动signer1，可运行以下指令：
```
geth --datadir ./data --networkid 55661 --port 2002 --unlock signer1_address console 
```

启动后，会进入一个console的界面，可以运行一些`eth.getBalance()`等指令来操作geth。
 ![这里写图片描述](http://img.blog.csdn.net/20171005110406330?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvY29kZV9zZWdtZW50/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

启动signer2 ，注意port和address都不一样，并且在signer2的目录下
 ![这里写图片描述](http://img.blog.csdn.net/20171005110422620?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvY29kZV9zZWdtZW50/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)
 
在node1的启动信息中有一段encode信息，下图已标黄，如下：
 ![这里写图片描述](http://img.blog.csdn.net/20171005110444965?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvY29kZV9zZWdtZW50/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)
 
由于目前是在私有链上，没有设定启动节点，也没设定static node，各节点启动后是无法相互通信的。所以，geth要连上对方的节点就必须先设置好`enode://<id>@<ip>:<port>`，复制刚刚启动node1时出现的enode信息，将[::]替换为127.0.0.1，这样就可以让其他节点加入。

在node2, signer1, signer2 的geth console 界面下分别运行如下指令：
 ![这里写图片描述](http://img.blog.csdn.net/20171005110734461?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvY29kZV9zZWdtZW50/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)
 ![这里写图片描述](http://img.blog.csdn.net/20171005110748261?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvY29kZV9zZWdtZW50/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)
 ![这里写图片描述](http://img.blog.csdn.net/20171005110804061?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvY29kZV9zZWdtZW50/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)


完成后，在node1的geth console输入`admin.peers`应该要看到三个节点资讯，如下：（每个id对应其他节点的encode字段信息）
 ![这里写图片描述](http://img.blog.csdn.net/20171005110827483?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvY29kZV9zZWdtZW50/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

###启动Miner
在signer1的console界面，键入`miner.start()`，geth就会开始挖矿了。在signer1 的console 会出现正在mining 的信息。
 ![这里写图片描述](http://img.blog.csdn.net/20171005110926675?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvY29kZV9zZWdtZW50/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)
 
其他节点（node1和node2）则会收到import block 的信息。如下：
 ![这里写图片描述](http://img.blog.csdn.net/20171005110947141?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvY29kZV9zZWdtZW50/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)
 
### 进行交易

在node1的console下，使用指令`eth.getBalance("<Address>")`来查询账户余额。
 ![这里写图片描述](http://img.blog.csdn.net/20171005111050114?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvY29kZV9zZWdtZW50/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)
 
使用`eth.sendTransaction`指令来把一些ether从node1转到node2。首先，需要将node1的account解锁，然后才能转账，如下所示：
 ![这里写图片描述](http://img.blog.csdn.net/20171005111106382?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvY29kZV9zZWdtZW50/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)
 
在signer1的console下，运行`miner.start()`，在几秒后就会看到一个含有一笔交易的block产出，其他结点也会收到信息：
 ![这里写图片描述](http://img.blog.csdn.net/20171005111130365?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvY29kZV9zZWdtZW50/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)
 ![这里写图片描述](http://img.blog.csdn.net/20171005111149866?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvY29kZV9zZWdtZW50/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast) 

查看node1和node2的余额，如下：
![这里写图片描述](http://img.blog.csdn.net/20171005111217194?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvY29kZV9zZWdtZW50/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

### 加入新的信任节点

在Clique 共识机制中是使用Clique提供的API来进行节点管理。

signer2 是一开始未设定在创世块中信任列表的节点，如果这时候让它启动miner，会遇到一个未授权的错误，如下：
 ![这里写图片描述](http://img.blog.csdn.net/20171005111258333?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvY29kZV9zZWdtZW50/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

必须回到已经在授权名单内的节点的console界面下，将新的节点加入。在signer1 的console 输入指令：
 ![这里写图片描述](http://img.blog.csdn.net/20171005111337438?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvY29kZV9zZWdtZW50/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

在signer2 的console，运行`miner.start()`发现功能未实现。
 ![这里写图片描述](http://img.blog.csdn.net/20171005111357460?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvY29kZV9zZWdtZW50/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

遇到这种情况，应将signer1和signer2都关闭，重新启动，如下：
重新启动signer1，输入账户密码，进入console界面
 ![这里写图片描述](http://img.blog.csdn.net/20171005111417553?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvY29kZV9zZWdtZW50/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)
重新启动signer2，输入账户密码，进入console界面
 ![这里写图片描述](http://img.blog.csdn.net/20171005111435106?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvY29kZV9zZWdtZW50/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

在signer2中运行`miner.start()`进行挖矿，提示Signed recently, must wait for others
 ![这里写图片描述](http://img.blog.csdn.net/20171005111449391?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvY29kZV9zZWdtZW50/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

在signer1中运行`miner.start()`进行挖矿，提示Signed recently, must wait for others
 ![这里写图片描述](http://img.blog.csdn.net/20171005111508559?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvY29kZV9zZWdtZW50/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

发现signer1和signer2都提示Signed recently, must wait for others，并且两个终端都没有继续挖矿，猜想可能是signer1和signer2之间没有建立通信，在signer1和signer2中分别运行admin.pees，发现都没有链接节点，如下：
 ![这里写图片描述](http://img.blog.csdn.net/20171005111529123?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvY29kZV9zZWdtZW50/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

因此在signer1中通过admin.addPeer()添加signer2，使二者可以通信，signer1和signer2都重新继续挖矿；其中signer1生成26号块后，signer2从刚才生成25号块提示Signed recently, must wait for others的地方Imported new chain segment，新导入的就是signer1生成的26号块，随后，signer1和signer2交替签名，依次挖矿，如下所示：
 ![这里写图片描述](http://img.blog.csdn.net/20171005111545418?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvY29kZV9zZWdtZW50/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)
![这里写图片描述](http://img.blog.csdn.net/20171005111614794?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvY29kZV9zZWdtZW50/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast) 

###其他的Tips：
在终端不关闭的情况下，重新启动geth进入console界面，通过admin.peers查看链接信息，发现之前添加的节点信息都存在；但是，关闭终端后，重新打开一个新的终端，启动geth进入console界面，通过admin.peers查看链接信息，会发现是空的，需要重新运行admin.addPeer()添加，这一点一定要注意！！！

通过`clique.discard(signer_address)`删除信任节点。通过`clique.propose(signer_address，true)`添加信任节点，通过实验发现，无法立即生效，还需要一半以上的信任节点的投票[2]，[这点已证实](https://github.com/Agzs/geth-pbft-study/wiki/pbft%E6%8A%95%E7%A5%A8%E9%80%89%E4%B8%BE%E6%96%B0%E7%9A%84%E6%8E%88%E6%9D%83%E8%8A%82%E7%82%B9(signer))。一般要在当前所有的信任节点的终端中都执行clique.propose()，并且在各信任节点的终端都同时挖矿，轮流挖出`len(signers)/2`个块后，新添加的信任节点就可已挖矿了，具体解决方法需要查看源代码，部分重要的函数已放到博客：http://blog.csdn.net/code_segment/article/details/78150163。

提示Signed recently, must wait for others的原因，通过查看最新版go-ethereum项目的源代码，发现consensus/clique/clique.go 的seal函数如下：
 ![这里写图片描述](http://img.blog.csdn.net/20171005111746531?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvY29kZV9zZWdtZW50/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)
snap.Recents为map[uint64]common.Address，保存最近签名的signer
满足number <  limit || seen > number-limit条件即输出Signed recently, must wait for others提示，由于signer之间没有建立通信，所以都卡在 <-stop 语句处

### 参考：

1、https://medium.com/taipei-ethereum-meetup/%E4%BD%BF%E7%94%A8-go-ethereum-1-6-clique-poa-consensus-%E5%BB%BA%E7%AB%8B-private-chain-1-4d359f28feff

2、https://ethereum.stackexchange.com/questions/15541/how-to-add-new-sealer-in-geth-1-6-proof-of-authority

