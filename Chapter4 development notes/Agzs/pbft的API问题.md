参考clique，设计了pbft的API，主要用于实现民主投票，添加新的VP(signer)

全部配置都和clique相同，在go-ethereum和geth-pbft中搜索了所有的clique关键字，然后逐一对照，逻辑完全没有问题，但是在console界面中分别运行clique和pbft，发现clique可以正常运行，但是pbft提示未定义，如下：
```go
...
INFO [11-20|22:28:08] Initialised chain configuration          config="{ChainID: 217 Homestead: 1 DAO: <nil> DAOSupport: false EIP150: 2 EIP155: 3 EIP158: 3 Metropolis: <nil> Engine: clique}"
...
 modules: admin:1.0 clique:1.0 debug:1.0 eth:1.0 miner:1.0 net:1.0 personal:1.0 rpc:1.0 txpool:1.0 web3:1.0

> clique
{
  proposals: {},
  discard: function(),
  getProposals: function(callback),
  getSigners: function(),
  getSignersAtHash: function(),
  getSnapshot: function(),
  getSnapshotAtHash: function(),
  propose: function()
}
```
```go
...
INFO [11-20|19:25:20] Initialised chain configuration          config="{ChainID: 9882 Homestead: 1 DAO: <nil> DAOSupport: false EIP150: 2 EIP155: 3 EIP158: 3 Metropolis: <nil> Engine: pbft}"
...
 modules: admin:1.0 debug:1.0 eth:1.0 miner:1.0 net:1.0 pbft:1.0 personal:1.0 rpc:1.0 txpool:1.0 web3:1.0

> pbft.
(anonymous): Line 2:1 Unexpected end of input
> pbft
ReferenceError: 'pbft' is not defined
    at <anonymous>:1:1
```

经过函数调用关系以及打印数据，发现有的打印信息会显示，而有的打印信息没有显示，初步猜想可能是文件问题，比如js文件或者json文件，通过查找只有一个web3.js文件有可疑，但是在文件中进行查找并没有发现和clique相关的语句，初步猜想错误

随后继续根据打印的信息进行查找，发现文件导入的包不同，比如修改了web3ext.go中的Modules变量，但是查找该变量的引用情况时发现并没有文件引用该变量，在ethereum/go-ethereum中进行搜索，发现console/console.go中有引用：
```go
if file, ok := web3ext.Modules[api]; ok {
...
}
```
但是，该变量导入的包是`"github.com/ethereum/go-ethereum/internal/web3ext"`，终于发现问题的根本原因了，由于geth-pbft中引用了ethereum/go-ethereum中的包，但是有的包中的文件是相同的，而修改的文件在geth-pbft目录中，程序编译运行时却用的是go-ethereum中的包，造成了不管如何修改geth-pbft目录中的文件，都没有影响到可执行程序。

解决方法：

在go-ethereum中查找所有引用web3ext的文件，发现主要集中在console目录下，如果直接在go-ethereum中进行修改的话，后期git push时不好操作，为方便项目编译，避免go-ethereum目录中文件的修改，所以采取以下方法：

在geth-pbft中添加console目录，该目录从ethereum/go-ethereum目录下拷贝而来，将在geth-pbft目录中所有涉及web3ext的文件中的包引用替换为`"github.com/yeongchingtarn/geth-pbft/internal/web3ext"`

此外，在geth-pbft目录中的所有文件，若存在和ethereum/go-ethereum相同的包（不单单是包名，还包括内部文件），必须替换为geth-pbft路径的包名

主要修改的文件有：

geth-pbft/console/console.go       
```go
"github.com/yeongchingtarn/geth-pbft/internal/jsre"    //=> ethereum/go-ethereum -> yeongchingtarn/geth-pbft
"github.com/yeongchingtarn/geth-pbft/internal/web3ext" //=> ethereum/go-ethereum -> yeongchingtarn/geth-pbft
```
geth-pbft/cmd/geth目录下的accountcmd.go、chaincmd.go、consolecmd.go、main.go中
```go
"github.com/yeongchingtarn/geth-pbft/console" //=> ethereum/go-ethereum -> yeongchingtarn/geth-pbft --Agzs 11.20
```

注意：
`geth-pbft/console/bridge.go`中需要引用`accounts/usbwallet`，虽然geth-pbft中含有该包名，但是内部文件和ethereum/go-ethereum中的文件不同，如果进行替换的话会造成一些变量未定义，所以该处不进行替换。

现在pbft的API可以使用，如下，但是尚未测试函数
```go
...
modules: admin:1.0 debug:1.0 eth:1.0 miner:1.0 net:1.0 pbft:1.0 personal:1.0 rpc:1.0 txpool:1.0 web3:1.0

> p
parseFloat parseInt pbft personal propertyIsEnumerable 
> pbft
{
  proposals: {},
  discard: function(),
  getProposals: function(callback),
  getSigners: function(),
  getSignersAtHash: function(),
  getSnapshot: function(),
  getSnapshotAtHash: function(),
  propose: function()
}
```
