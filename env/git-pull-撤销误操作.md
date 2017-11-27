### git pull 撤销误操作
本来想把github上的newpbft合并到本地的newpbft分支上，由于没有查看当前分支，直接运用`git pull origin newpbft`，结果将newpbft合并到了master分支中。

#### 解决方法

1、运行`git reflog`命令查看你的历史变更记录，如下：
```
fdb70fe HEAD@{0}: pull origin newpbft: Fast-forward
40a9a83 HEAD@{1}: checkout: moving from guan to master
b3fa4c3 HEAD@{2}: commit: copy from newpbft, first init
71bf0ec HEAD@{3}: checkout: moving from newpbft to guan
71bf0ec HEAD@{4}: commit: 1. add moveStore() to clean up certStore and blockStore.
1006d67 HEAD@{5}: commit: 1. Add PBFT branch to Puppeth.
fa3fb56 HEAD@{6}: commit: 1. change some errors about packages and vars
5f40fdc HEAD@{7}: checkout: moving from master to newpbft
40a9a83 HEAD@{8}: clone: from https://github.com/yeongchingtarn/geth-pbft.git
```

2、然后用`git reset --hard HEAD@{n}`，（n是你要回退到的引用位置）回退。

比如上图可运行 `git reset --hard 40a9a83`
