## golang 安装使用 protobuf 的教程

### 1、下载protobuf的编译器protoc

[下载地址：https://github.com/google/protobuf/releases](https://github.com/google/protobuf/releases)
#### window：
 * 下载: protoc-3.3.0-win32.zip
 * 解压，把bin目录下的protoc.exe复制到GOPATH/bin下，GOPATH/bin加入环境变量。
 * 当然也可放在其他目录，需加入环境变量，能让系统找到protoc.exe
#### linux：
 * 下载：`protoc-3.3.0-linux-x86_64.zip` 
 * 将压缩包解压，把`protoc-3.3.0-linux-x86_64/bin`目录下的`protoc`复制到`$GOPATH/bin`下
 * `$GOPATH/bin`加入环境变量。编辑`~/.profile`文件或`~/.bashrc`文件，二者不同可见[GOPATH设置](http://blog.csdn.net/code_segment/article/details/77376270),在最后添加：
 * `export GOPATH="go项目的路径"`
 * `export PATH=$PATH:$GOPATH/bin`
 * 如果喜欢编译安装的，也可下载源码自行安装，最后将可执行文件加入环境变量。
    源码地址：https://github.com/google/protobuf

### 2、获取protobuf的编译器插件protoc-gen-go
 * 进入GOPATH目录
 * 运行 `go get -u github.com/golang/protobuf/protoc-gen-go`
 * 如果成功，会在GOPATH/bin下生成protoc-gen-go.exe文件

### 3、拷贝库文件
* 从解压目录`protoc-3.3.0-linux-x86_64/include/google/protobuf`中拷贝全部文件,复制到 `GOPATH/src/google/protobuf`目录下

### 4、运行
 * `protoc --go_out=. *.proto`
 * Note，`*`前有空格，`*`是你proto文件的名称，更多详细内容，请阅[官方文档](https://github.com/golang/protobuf)
    


