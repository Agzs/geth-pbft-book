step 1 安装lantern lantern下载地址.[Lantern](https://github.com/getlantern/forum/issues/833)

step 2 编辑文件如.bash_profile，输入如下内容：

    export http_proxy=http://127.0.0.1:60836
    export https_proxy=$http_proxy
    export ftp_proxy=$http_proxy
    export rsync_proxy=$http_proxy
    export no_proxy="localhost,127.0.0.1,.dade.com"

（windows下使用set http_proxy=127.0.0.1:60836）

然后执行source .bash_profile即可生效。

-- 以上内容请根据自己的实际情况做调整。目前lantern的代理端口是60000
注意端口一定要修改正确！打开蓝灯页面左边的设置，查看http代理端口是多少，设置好了才可使用。


step3 试一下 env
检查下是否有刚才添加的信息。

Dade-MBP:go-demo dade$ env
......
http_proxy=http://127.0.0.1:60836
......
ftp_proxy=http://127.0.0.1:60836
......
rsync_proxy=http://127.0.0.1:60836

step4: 现在可以欢快的进行go get 了

原文转自https://dade.io/archives/116/


### run with no UI: -headless