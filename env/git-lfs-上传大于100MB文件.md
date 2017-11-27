github允许上传的文件上限为100MB，如果上传的文件过大，会提示以下错误：
```go
ethtest@ethtest:~/application$ git push origin master
Username for 'https://github.com': Agzs
Password for 'https://Agzs@github.com': 
Counting objects: 3, done.
Delta compression using up to 8 threads.
Compressing objects: 100% (2/2), done.
Writing objects: 100% (3/3), 279.91 MiB | 1.81 MiB/s, done.
Total 3 (delta 0), reused 0 (delta 0)
remote: error: GH001: Large files detected. You may want to try Git Large File Storage - https://git-lfs.github.com.
remote: error: Trace: 2c725d84981f341d33e6ee5446bf66e3
remote: error: See http://git.io/iEPt8g for more information.
remote: error: File gopath.zip is 286.61 MB; this exceeds GitHub's file size limit of 100.00 MB
To https://github.com/Agzs/gopath.git
 ! [remote rejected] master -> master (pre-receive hook declined)
error: failed to push some refs to 'https://github.com/Agzs/gopath.git'
```

解决方法：

安装git的辅助程序git-lfs

### 1、下载

下载链接：https://github.com/git-lfs/git-lfs/releases/download/v2.3.4/git-lfs-linux-amd64-2.3.4.tar.gz

或命令行运行：`wget https://github.com/git-lfs/git-lfs/releases/download/v2.3.4/git-lfs-linux-amd64-2.3.4.tar.gz`

### 2、解压
```bash
tar -xzf git-lfs-linux-amd64-2.3.4.tar.gz
```

### 3、运行.sh文件

命令行打开install.sh文件所在目录，运行`sudo ./install.sh`

### 4、使用

Now, it's time to add some large files to a repository. The first step is to
specify file patterns to store with Git LFS. These file patterns are stored in
`.gitattributes`.

```bash
$ mkdir large-repo
$ cd large-repo
$ git init

# Add all zip files through Git LFS
$ git lfs track "*.zip"
```

Now you're ready to push some commits:

```bash
$ git add .gitattributes
$ git add my.zip
$ git commit -m "add zip"
```

You can confirm that Git LFS is managing your zip file:

```bash
$ git lfs ls-files
my.zip
```

Once you've made your commits, push your files to the Git remote:

```bash
$ git remote add origin https://github.com/Agzs/gopath.git
$ git push origin master
Sending my.zip
LFS: 12.58 MB / 12.58 MB  100.00 %
Counting objects: 2, done.
Delta compression using up to 8 threads.
Compressing objects: 100% (5/5), done.
Writing objects: 100% (5/5), 548 bytes | 0 bytes/s, done.
Total 5 (delta 1), reused 0 (delta 0)
To https://github.com/git-lfs/git-lfs-test
   67fcf6a..47b2002  master -> master
```

### 5、clone到本地

```bash
git lfs clone https://github.com/Agzs/gopath.git
```

### 6、Need Help?

You can get help on specific commands directly:

```bash
$ git lfs help <subcommand>
```

参考：https://github.com/git-lfs/git-lfs


