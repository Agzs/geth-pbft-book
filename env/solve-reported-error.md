### import cycle not allowed
golang不允许循环导包,如果检测到import cycle，会在编译时报错，通常import cycle是因为设计错误或包的规划问题。
```go
import(
"package A"
"package B"
)
```
* 如果package A中已经导入package B，而本package中又导入package B
* 或者 package A依赖package B,同时 package B 依赖package A

比如在consensus/pbft/pbft.go中导入geth-pbft/eth包，在geth-pbft/eth/backend.go中导入consensus/pbft

这样就会在编译时报 "import cycle not allowed"。

如何避免重复导入包的问题，就需要在设计时规划好包。

### go
* 在某个包中定义的函数，若函数名开头为大写字母，则为public类型，可在其他包中被调用。
* 定义的结构体成员变量一般为私有变量，可通过自定义GetXXX()或SetXXX()进行成员变量的修改。
