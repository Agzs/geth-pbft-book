##
### 本地仓库上传
* 1、`git status`查看当前修改的文件
* 2、`git add --all`
* 3、`git commit -m "修改的内容等相关信息"
* 4、`git push origin yourbranch`

### 创建分支
* 创建分支：`git checkout -b yourbranch`
* 切换分支：`git checkout yourbranch` 
* 删除分支：`git branch -d yourbranch` //在其他分支下删除yourbranch分支，不能在yourbranch分支下删除yourbranch分支

### 合并分支
* 查看当前分支：`git branch`
* 切换到目标分支`newpbft`：`git checkout newpbft`，保证是最新的newpbft分支
* 将其他分支`guan`合并到目标分支`newpbft`：`git merge --no-ff -m "merge的相关信息，类似commit" guan` //在newpbft分支下进行
* 若提示冲突，找到冲突文件，选择性保存，具体参考[解决分支冲突](https://www.liaoxuefeng.com/wiki/0013739516305929606dd18361248578c67b8067c8c017b000/001375840202368c74be33fbd884e71b570f2cc3c0d1dcf000)
* 修改完冲突后，`git push origin newpbft`

### 参考：
#### 廖雪峰讲git

[分支管理](https://www.liaoxuefeng.com/wiki/0013739516305929606dd18361248578c67b8067c8c017b000/0013743862006503a1c5bf5a783434581661a3cc2084efa000)

[创建与合并分支](https://www.liaoxuefeng.com/wiki/0013739516305929606dd18361248578c67b8067c8c017b000/001375840038939c291467cc7c747b1810aab2fb8863508000)

[解决分支冲突](https://www.liaoxuefeng.com/wiki/0013739516305929606dd18361248578c67b8067c8c017b000/001375840202368c74be33fbd884e71b570f2cc3c0d1dcf000)

#### git官方数目
[分支创建与合并](https://git-scm.com/book/zh/v2/Git-%E5%88%86%E6%94%AF-%E5%88%86%E6%94%AF%E7%9A%84%E6%96%B0%E5%BB%BA%E4%B8%8E%E5%90%88%E5%B9%B6)