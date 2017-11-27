## 安装librocksdb.so.4.1的共享库
注：以下命令需在root模式下进行

### 1、clone rocksDB
命令行运行`git clone https://github.com/facebook/rocksdb.git`

### 2、切换到4.1.fb分支，编译
```bash
cd rocksdb
git checkout 4.1.fb
make shared_lib
```

### 3、拷贝/usr/local/lib目录
```bash
cp librocksdb.so /usr/local/lib
cp librocksdb.so.4 /usr/local/lib
cp librocksdb.so.4.1 /usr/local/lib
cp librocksdb.so4.1.0 /usr/local/lib
```

### 4、配置新共享库目录
共享库文件安装到了/usr/local/lib(很多开源的共享库都会安装到该目录下)或其它"非/lib或/usr/lib"目录下, 那么在执行ldconfig命令前, 还要把新共享库目录加入到共享库配置文件/etc/ld.so.conf中, 如下:
```bash
# cat /etc/ld.so.conf
include ld.so.conf.d/*.conf
# echo "/usr/local/lib" >> /etc/ld.so.conf
```

### 5、运行ldconfig
```bash
# ldconfig
```
可能会提示`/sbin/ldconfig.real: /usr/local/lib/librocksdb.so.4.1 is not a symbolic link`，但是`geth version`已经可以正常运行了

### 参考

[共享库错误](http://blog.chinaunix.net/uid-26212859-id-3256667.html)

[XX is not a symbolic link](https://stackoverflow.com/questions/11542255/ldconfig-error-is-not-a-symbolic-link)