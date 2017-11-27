## Go interface list

可以使用list-interfaces工具查看某个package中的代码实现了那些接口：

~/gopath/bin/list-interfaces --codedir /root/gopath/src/github.com/hyperledger/fabric/consensus/ --gopath /root/gopath/ --outputfile result

具体list-interfaces工具下载安装自行google。


## golang 继承和嵌入
### 参考http://blog.csdn.net/nature19862001/article/details/72858243
如果struct继承了一个匿名struct，其会继承所有成员和函数，调用函数时会调用子类的函数，如果子类没有则调用父类。
如果要明确调用父类的变量和函数，则需明确使用.父类.函数或指针。


## brew install tips:
### Install an older version of a package

https://www.client9.com/using-macos-homebrew-to-install-a-specific-version/

git -C "$(brew --repo homebrew/core)" fetch --unshallow

git -C "$(brew --repo homebrew/core)" log master -- Formula/phantomjs.rb

BREWURL=https://raw.githubusercontent.com/Homebrew/homebrew-core

brew install ${BREWURL}/HASH/Formula/NAME.rb

Note: replace NAME and HASH with appropriate values.


## interfaces.go error: use go1.8 or higher

## For memo:


```
To use the bundled libc++ please add the following LDFLAGS:
  LDFLAGS="-L/usr/local/opt/llvm/lib -Wl,-rpath,/usr/local/opt/llvm/lib"

This formula is keg-only, which means it was not symlinked into /usr/local,
because macOS already provides this software and installing another version in
parallel can cause all kinds of trouble.

If you need to have this software first in your PATH run:
  echo 'export PATH="/usr/local/opt/llvm/bin:$PATH"' >> ~/.bash_profile

For compilers to find this software you may need to set:
    LDFLAGS:  -L/usr/local/opt/llvm/lib
    CPPFLAGS: -I/usr/local/opt/llvm/include


If you need Python to find bindings for this keg-only formula, run:
  echo /usr/local/opt/llvm/lib/python2.7/site-packages >> /usr/local/lib/python2.7/site-packages/llvm.pth
  mkdir -p /Users/zhiguo/Library/Python/2.7/lib/python/site-packages
  echo 'import site; site.addsitedir("/usr/local/lib/python2.7/site-packages")' >> /Users/zhiguo/Library/Python/2.7/lib/python/site-packages/homebrew.pth
==> Summary
🍺  /usr/local/Cellar/llvm/5.0.0: 2,478 files, 1.2GB
```