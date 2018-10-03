# vps-
首先，我是准备将谷歌云盘先挂载到我的阿里云上，下面是copy摘抄的教程，嘻嘻

Rclone挂载

安装EPEL源：

    yum -y install epel-release
    
安装一些基本组件和依赖：

    yum -y install wget unzip screen fuse fuse-devel
    
下载Rclone解压然后进入目录：

    cd /root && wget https://downloads.rclone.org/v1.42/rclone-v1.42-linux-amd64.zip
    unzip rclone-v1.42-linux-amd64.zip
    cd rclone-v1.42-linux-amd64
    
运行Rclone开始配置：

    ./rclone config
    
第一步选择n，然后回车输入一个name，建议这个name设置的简单好记一点，如图所示：
![Image text](https://raw.githubusercontent.com/suiyuan2012/img-folder/master/3494175083.png）
然后选择我们要挂载的类型，这里选择11，切记要选对了：

接着client_id、client_secret、service_account_file都留空直接回车，Scope that rclone should use when requesting access from drive.Choose a number from below, or type in your own value这里选择1，root_folder_id留空回车，Use auto config?这里我们选择n，如图所示：

lala.im_2018-03-14-20-5013.png
现在rclone会在终端内给我们回显一个GoogleDrive的授权登录地址，如图所示：

lala.im_2018-03-14-08-7814.png
我们复制这个地址然后用本地电脑的浏览器打开并登录（需翻墙），然后点击允许按钮，接着复制如下图所示的授权代码：

lala.im_2018-03-14-25-204.png
回到终端内粘贴授权代码然后回车，继续按如下图操作，依次输入n、y、q：

全部完成后，现在新建一个你要挂载的目录：

mkdir -p /marisn/gdrive
先把rclone的可执行文件复制到/usr/bin：

cp /root/rclone-v1.42-linux-amd64/rclone /usr/bin/rclone
新建一个rclone.service文件：

vi /usr/lib/systemd/system/rclone.service
写入：

[Unit]
Description=rclone
    
[Service]
User=root
ExecStart=/usr/bin/rclone mount marisn: /marisn/gdrive --allow-other --allow-non-empty --vfs-cache-mode writes
Restart=on-abort
    
[Install]
WantedBy=multi-user.target
重载daemon，让新的服务文件生效：

systemctl daemon-reload
现在就可以用systemctl来启动rclone了：

systemctl start rclone
设置开机启动：

systemctl enable rclone
停止、查看状态可以用：

systemctl stop rclone
systemctl status rclone
重启你的VPS，然后查看一下rclone的服务起来没，接着查看一下盘子挂上去没：

reboot
systemctl status rclone
df -h
哎呀妈呀，才把挂载copy完就这么多了，接下来讲如何利用aria2离线下载自动上传到谷歌云盘

装依赖和组件：

yum -y install wget screen unzip gcc gcc-c++ openssl-devel
安装Aria2（CentOS6要升级GCC，7可以直接装）：

cd /root
wget https://github.com/aria2/aria2/releases/download/release-1.33.1/aria2-1.33.1.tar.gz
tar xzvf aria2-1.33.1.tar.gz
cd aria2-1.33.1
./configure
make
make install
Aria2安装好后，我们在root目录下新建一个脚本文件，命名为autoupload.sh：

vi /root/autoupload.sh
写入：

#!/bin/bash
path=$3
downloadpath='/root/downloads' #此处/root/downloads为aria2下载时默认下载目录 自行更改
if [ $2 -eq 0 ]
        then
                exit 0
fi
while true; do
filepath=$path
path=${path%/*}; 
if [ "$path" = "$downloadpath" ] && [ $2 -eq 1 ]
    then
    rclone move "$filepath" /marisn/gdrive    #你设置的谷歌挂载目录 自行更改
    exit 0
elif [ "$path" = "$downloadpath" ]
    then
    mv "$filepath"/ /marisn/gdrive/"${filepath##*/}"/  #你设置的谷歌挂载目录 自行更改
    exit 0
fi
done
给脚本执行权限：

chmod +x /root/autoupload.sh
现在我们就可以启动aria2了：

aria2c --enable-rpc --rpc-listen-all --rpc-allow-origin-all --rpc-secret=marisn --on-download-complete=/root/autoupload.sh -c --dir /root/downloads -D
–rpc-secret=后面的值务必修改复杂一点，这是你的rpc连接密码！如果不设置或这个值设置的很容易被人猜到，会出现严重安全问题。

–on-download-complete=后面要指定执行脚本的路径，如果你的脚本路径不一样这里要做相应更改。

–dir后面的下载路径务必要和之前脚本内的下载路径一致。

将aria2加入自启【如果你的centos支持的话】：

echo "aria2c --enable-rpc --rpc-listen-all --rpc-allow-origin-all --rpc-secret=marisn --on-download-complete=/root/autoupload.sh -c --dir /root/downloads -D" >> /etc/rc.local
chmod +x /etc/rc.d/rc.local
现在来安装AriaNG。

先要安装一个Nginx：

vi /etc/yum.repos.d/nginx.repo
写入：

[nginx]
name=nginx repo
baseurl=http://nginx.org/packages/centos/7/$basearch/
gpgcheck=0
enabled=1
用yum安装：

yum -y install nginx
进入到Nginx的默认站点目录：

cd /usr/share/nginx/html
下载AriaNG并解压：

wget https://github.com/mayswind/AriaNg/releases/download/0.4.0/aria-ng-0.4.0.zip
unzip aria-ng-0.4.0.zip
注意有一个同名文件直接按y覆盖就行

启动Nginx：

systemctl start nginx
现在打开你的VPS公网IP就能访问到AriaNG了。先填写rpc连接密码，让AriaNG和Aria2连接上

注意：如果你不想单纯的浪费这台机子，可以像我一样搭建宝塔面板，然后AriaNG部分参考 [教程]Aria2离线下载+H5ai在线观看 中的教程

OK，再让我们来安装一个filebrowser，用来管理我们的网盘文件和实现在线播放视频等需求。

下载并解压filebrowser：

cd /root
wget https://github.com/filebrowser/filebrowser/releases/download/v1.9.0/linux-amd64-filebrowser.tar.gz
tar -zxvf linux-amd64-filebrowser.tar.gz
写个服务，让filebrowser开机启动。

先复制可执行文件到/usr/bin：

cp /root/filebrowser /usr/bin/filebrowser
新建一个服务文件：

vi /usr/lib/systemd/system/filebrowser.service
写入：

[Unit]
Description=filebrowser
    
[Service]
User=root
ExecStart=/usr/bin/filebrowser --port 2333
Restart=on-abort
    
[Install]
WantedBy=multi-user.target
其中2333为端口 如果是宝塔需要在安全里放行，还有6800端口，为aria2需要使用的端口，也要放行。

管理命令：

systemctl enable filebrowser
systemctl start filebrowser
systemctl status filebrowser
systemctl restart filebrowser
systemctl stop filebrowser
注：1是开机启动，2是现在运行，3是查看运行状态，4是重启，5是停止运行。

默认的管理员账号密码都是admin，登录进去的第一件事就是把密码修改掉。接着我们要改变一下目录指定的路径为刚刚设置的挂载盘/marisn/gdrive。

整了半天，终于整蛊完事了

说说感想吧，1台国外服务器+1个无限云盘=so many 小姐姐

挂载后，可以不FQ直接在线看，我们一般都知道谷歌家的都要FQ才能访问，但是如果你有国外服务器，就相当于从你国外服务器转到谷歌，但是比较费流量，必须整个大带宽多流量的机子来玩。

后续再说说吧，那个filebrowser有些视频不支持在线直接从它那里面看，小技巧就是复制下载链接，拷贝到

https://tools.67cc.cn/m3u8.php?url=[视频地址]
Snipaste_2018-07-30_10-29-59.png
2018年7月30日18:02:54

再补充一些技巧：

假如你选择搭建宝塔，可以直接反代filebrowser，加个CF的盾，就可以分享给小伙伴一起看了。

什么？你不是很懂反代？

那接下来我给你简单演示一下，字不重要看图

Snipaste_2018-07-30_18-06-54.png
在使用过程中，我觉得filebrowser看小姐姐不安逸，我还是换一个WEB文件浏览器吧

宝塔新建网站，下载KODExplorer到目录

https://kodcloud.com/download/

解压后安装即可

由于宝塔默认开启防跨站，只需要关闭即可。

Snipaste_2018-07-30_18-41-53.png
这样你才能访问设置的谷歌挂载盘

Snipaste_2018-07-30_18-37-15.png
