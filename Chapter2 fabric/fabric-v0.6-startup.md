## `fabric/peer/main.go`

`main.go`包含全局变量`mainCmd`, 使用`cobra.Command`赋值，详细流程暂不考虑. 
```go
// The main command describes the service and defaults to printing the help message.
var mainCmd = &cobra.Command{
	Use: "peer",
	PersistentPreRunE: func(cmd *cobra.Command, args []string) error {
		peerCommand := getPeerCommandFromCobraCommand(cmd)
		flogging.LoggingInit(peerCommand)

		return core.CacheConfiguration()
	},
	Run: func(cmd *cobra.Command, args []string) {
		if versionFlag {
			version.Print()
		} else {
			cmd.HelpFunc()(cmd, args)
		}
	},
}
```
`mainCmd`的使用主要在`func main()`中，主要涉及两种操作：`mainCmd.AddCommand()` 和` mainCmd.Execute()`，如下：
```go
func main() {
	...
	mainCmd.AddCommand(version.Cmd())
	mainCmd.AddCommand(node.Cmd())
	mainCmd.AddCommand(network.Cmd())
	mainCmd.AddCommand(chaincode.Cmd())

	...
	if mainCmd.Execute() != nil {
		os.Exit(1)
	}
	logger.Info("Exiting.....")
}
```

`mainCmd`将`version`、`node`、`network`、`chaincode`相关的命令作为自己的子命令添加到命令树中，在`main()`函数的最后通过调用`mainCmd.Execute()`，进一步调用`mainCmd.ExecuteC()`，递归执行`mainCmd`的所有子命令，具体的函数定义在`github.com/spf13/cobra/command.go`中。

###  `version.Cmd()`

`peer/version/version.go`中定义了`func Cmd()`. 如下:

```go
// Cmd returns the Cobra Command for Version
func Cmd() *cobra.Command {
	return cobraCommand
}
```
`version.Cmd()`比较简单，对比其他三个`(node、network、chaincode)`，该函数只是返回一个构造好的`cobraCommand`，唯一不同的是，该`cobraCommand`中的`Run`成员变量自定义为print()函数，用于输出当前的可执行版本号。

### `node.Cmd()`

`peer/node/node.go`中定义了`func Cmd()`. 如下:

```go
func Cmd() *cobra.Command {
	nodeCmd.AddCommand(startCmd())
	nodeCmd.AddCommand(statusCmd())
	nodeCmd.AddCommand(stopCmd())

	return nodeCmd
}
```
在`node.Cmd()`函数中`nodeCmd`将`startCmd()`、`statusCmd()`、`stopCmd()`作为自己的子命令添加到命令树中，其中`startCmd()`和项目的启动息息相关，在此只简单提及，后续进一步讨论。

*  `startCmd()`

`peer/node/start.go`中定义了`func startCmd()`. 如下:

```go
func startCmd() *cobra.Command {
	// Set the flags on the node start command.
	flags := nodeStartCmd.Flags()
	flags.BoolVarP(&chaincodeDevMode, "peer-chaincodedev", "", false,
		"Whether peer in chaincode development mode")

	return nodeStartCmd
}
```

函数返回值为全局变量，在同一文件定义如下：

```go
var nodeStartCmd = &cobra.Command{
	Use:   "start",
	Short: "Starts the node.",
	Long:  `Starts a node that interacts with the network.`,
	RunE: func(cmd *cobra.Command, args []string) error {
		return serve(args)
	},
}
```

其中成员变量`RunE`为自定义函数，调用同一文件下的`func serve()`函数，至此，fabric开始初始化诸如`engine`、`peer`、`peerServer`等实体。

*  `statusCmd()`

`peer/node/status.go`中定义了`func statusCmd()`. 

函数返回值同样为同一文件下的`nodeStatusCmd`变量，其成员变量`Run`调用同一文件下的`status()`函数，打印`serverClient`的状态信息。

函数的调用模式同`startCmd()`。

*  `stopCmd()`

`peer/node/stop.go`中定义了`func stopCmd()`. 

函数返回值同样为同一文件下的`nodeStopCmd`变量，其成员变量`Run`调用同一文件下的`stop()`函数，通过`gRPC`关闭`serverClient`和`database`。

函数的调用模式同`startCmd()`。

###  `network.Cmd()`

`peer/network/network.go`中定义了`func Cmd()`. 如下:
```go
func Cmd() *cobra.Command {
	networkCmd.AddCommand(loginCmd())
	networkCmd.AddCommand(listCmd())

	return networkCmd
}
```

* `loginCmd()`
`peer/network/login.go`中定义了`func loginCmd()`
函数返回值同样为同一文件下的` networkLoginCmd`变量，其成员变量`RunE`（和之前的Run相同，只是返回值多了一个error类型）调用同一文件下的`login()`函数，该函数和PKI相关，官方注释如下:
>login() confirms the enrollmentID and secret password of the client with the CA 
and stores the enrollment certificate and key in the Devops server.

* `listCmd()`
`peer/network/list.go`中定义了`func listCmd()`
函数返回值同样为同一文件下的`networkListCmd`变量，其成员变量`RunE`调用同一文件下的`networkList()`函数，该函数官方注释如下:
>Show a list of all existing network connections for the target peer node,
includes both validating and non-validating peers

###  `chaincode.Cmd()`

`peer/chaincode/chaincode.go`中定义了`func Cmd()`. 如下:
```go
func Cmd() *cobra.Command {
	...
	chaincodeCmd.AddCommand(deployCmd())
	chaincodeCmd.AddCommand(invokeCmd())
	chaincodeCmd.AddCommand(queryCmd())

	return chaincodeCmd
}
```
* deployCmd()
`peer/chaincode/deploy.go`中定义了`func deployCmd()`. 
函数返回值同样为同一文件下的`chaincodeDeployCmd`变量，其成员变量RunE调用同一文件下的`chaincodeDeploy()`函数，用于部署`chaincode`，如下：
```go
func chaincodeDeploy(cmd *cobra.Command, args []string) error {
	spec, err := getChaincodeSpecification(cmd)
	...
	devopsClient, err := common.GetDevopsClient(cmd)
	...
	chaincodeDeploymentSpec, err := devopsClient.Deploy(context.Background(), spec)
	...
	logger.Infof("Deploy result: %s", chaincodeDeploymentSpec.ChaincodeSpec)
	fmt.Printf("Deploy chaincode: %s\n", chaincodeDeploymentSpec.ChaincodeSpec.ChaincodeID.Name)

	return nil
}
```
* invokeCmd()
`peer/chaincode/invoke.go`中定义了`func invokeCmd()`. 
函数返回值同样为同一文件下的`chaincodeInvokeCmd`变量，其成员变量RunE调用同一文件下的`chaincodeInvoke()`函数，该函数又进一步调用`chaincodeInvokeOrQuery()`，`chaincodeInvokeOrQuery()`定义在peer/chaincode/common.go中，官方注释如下：
> chaincodeInvokeOrQuery invokes or queries the chaincode. 
If successful, the INVOKE form prints the transaction ID on STDOUT, 
and the QUERY form prints the query result on STDOUT. 

* queryCmd()
`peer/chaincode/query.go`中定义了`func queryCmd()`. 
函数返回值同样为同一文件下的`chaincodeQueryCmd`变量，其成员变量`RunE`调用同一文件下的`chaincodeQuery()`函数，该函数又进一步调用`chaincodeInvokeOrQuery()`，即 `queryCmd()`和`invokeCmd()`最终调用同一个函数，通过`chaincodeInvokeOrQuery()`函数的第三个`bool`类型的参数区分指令。

### 补充
`version`、`node`、`network`、`chaincode`各自的`Cmd()`函数都会返回 `XXXCmd`（其中`XXX`代表`network`、`node`等）， `XXXCmd`也是通过`cobra.Command`实现的，只不过`version`的`Cmd`比较简单，在`cobraCommand`中添加一个`print()`函数就实现了其功能，而另外三个仍然需要继续为自己的命令树添加子命令，各命令初始化赋值通用格式如下：
```go
var XXXCmd = &cobra.Command{
	Use:   XXXFuncName,
	Short: fmt.Sprintf("%s specific commands.", XXXFuncName),
	Long:  fmt.Sprintf("%s specific commands.", XXXFuncName),
}
```
其他命令的初始化也大同小异，只是单纯的成员变量的增删。

## Summary
在`func main()`中, 通过`mainCmd.AddCommand(XXX.Cmd())` 逐步按功能添加各自的子命令，形成一个命令树;后期通过调用`mainCmd.Execute()` 递归执行各子命令，比较重要的就是，node.Cmd()最终会调用`peer/node/start.go` 文件下的`func serve()`函数，fabric开始初始化诸如`engine`、`peer`、`peerServer`等实体。详细过程后期会进一步分析。

命令树如下：
![这里写图片描述](http://img.blog.csdn.net/20171008172530911?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvY29kZV9zZWdtZW50/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)


## 附录

### Cobra
Cobra既是创建强大的现代CLI(Command Line Applications )应用程序的库，也是用于生成应用程序和命令文件的程序。
* Github源码：https://github.com/spf13/cobra
* Cobra简介：下载源码后github.com/spf13/viper 目录下的Readme.md文件
* ubuntu16.04下配置指令： `$ go get github.com/spf13/cobra`
* 具体使用：http://studygolang.com/articles/7588

### viper
开源配置解决方案，满足不同的对配置文件的使用的要求，fabric采用viper来解决配置问题
* Github源码：https://github.com/spf13/viper
* Viper简介：下载源码后github.com/spf13/viper 目录下的Readme.md文件
* ubuntu16.04下配置指令： `$ go get github.com/spf13/viper`
* 具体使用：http://studygolang.com/articles/4244