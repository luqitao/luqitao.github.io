---
layout: post
title: docker集成criu实现热迁移功能的使用方法
tags: docker criu migrate
---

# 软件版本要求
Docker要管理运行在内部的所有containers，因此CRIU需要在Docker内运行，而不是单独运行。
组件 | 版本
---|---
docker | ce=17.03+ ee=1.13+
criu | 2.0+
centos | 7.3
kernel | 3.10.0-957

# criu安装
第一种方式，通过yum源安装
```
# yum install criu -y
```
第二种方式，通过源码安装
```
1.下载源码
# git clone https://github.com/checkpoint-restore/criu.git
2.安装编译所需的依赖包
# yum install gcc protobuf protobuf-c protobuf-c-devel protobuf-compiler protobuf-devel protobuf-python libnet-devel libnl3-devel libcap-devel  asciidoc xmlto -y
3.编译源码
# cd criu
# make install
4.测试，无报错信息表示安装成功
# criu check
Looks good.
```
# docker安装
```
1.卸载旧版本
# yum remove docker docker-common docker-selinux docker-engine -y
2.配置docker-ce镜像仓库
安装依赖包
# yum install -y yum-utils device-mapper-persistent-data lvm2
添加yum源
# yum-config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo
3.安装最新版Docker CE
# yum makecache fast
# yum install docker-ce -y
4.启动Docker
# systemctl enable docker
# systemctl start docker
5.开启热迁移功能
修改配置文件
# echo "{\"experimental\": true}" >> /etc/docker/daemon.json
# systemctl restart docker
```
# 进程热迁移
本机热迁移的过程不做描述，因此需要准备两台主机，下面docker进程的热迁移也一样
## 准备一个程序
criu提供了一个go-binding的库叫go-criu用来完成进程的热迁移，首先下载源码到GOPATH目录下
```
# git clone https://github.com/checkpoint-restore/go-criu.git
# cd github.com/checkpoint-restore/go-criu
```
2.编译

如果go环境配置没问题，直接执行make等待编译完成
```
# make
```
make过程中已经在本机执行了一次测试用例，我们主要关注跨主机的热迁移功能

3.运行测试程序

为了方便演示，可以将演示程序piggie以及criu测试程序test这两个文件cp到home目录下
```
# cp test/piggie test/test /home && cd /home
# ./piggie piggie.log
Child forked, pid 27636
# ps -ef | grep piggie
root     27636     1  0 10:17 ?        00:00:00 ./piggie piggie.log
root     27676  7298  0 10:17 pts/0    00:00:00 grep --color=auto piggie
# tail -f piggie.log
0
1
2
3
4
5
```
可以看到日志文件piggie.log中在不停的打印递增的数字
## 创建checkpoint
ssh到源主机node1上操作
```
# cd /home
# mkdir -p image
test程序的参数含义可以直接看源码checkpoint-restore\go-criu\test\main.go
# ./test dump `pidof piggie` image
CRIU version 31200
Dumping
TEST PRE DUMP
Success
创建完成后进程被杀掉了
# ps -ef | grep piggie
root     28478  7298  0 10:42 pts/0    00:00:00 grep --color=auto piggie
进程相关的数据保存在image目录下
# ll image/
total 224
-rw-r--r-- 1 root root   1762 Jan 24 10:42 core-1.img
-rw------- 1 root root  74269 Jan 24 10:42 dump.log
-rw-r--r-- 1 root root     44 Jan 24 10:42 fdinfo-2.img
-rw-r--r-- 1 root root    352 Jan 24 10:42 files.img
-rw-r--r-- 1 root root     18 Jan 24 10:42 fs-1.img
-rw-r--r-- 1 root root     32 Jan 24 10:42 ids-1.img
-rw-r--r-- 1 root root     40 Jan 24 10:42 inventory.img
-rw-r--r-- 1 root root    809 Jan 24 10:42 mm-1.img
-rw-r--r-- 1 root root    216 Jan 24 10:42 pagemap-1.img
-rw-r--r-- 1 root root 106496 Jan 24 10:42 pages-1.img
-rw-r--r-- 1 root root     22 Jan 24 10:42 pstree.img
-rw-r--r-- 1 root root     12 Jan 24 10:42 seccomp.img
-rw-r--r-- 1 root root     41 Jan 24 10:42 stats-dump
```
## 复制镜像文件到目的节点
从源节点将进程相关的数据拷贝到目的节点node2上，注意进程相关的文件都要拷过去且保持目录不变，日志文件piggie.log一定要使用创建checkpoint时间点的文件
```
# scp -r /home/image/ /home/piggie /home/piggie.log /home/test node2:/home
```
## 恢复
登录到node2节点上
```
# cd /home
# ./test restore image
CRIU version 31200
Restoring
Success
```
## 查看状态
可以看到进程会接着上次创建checkpoint的序号继续打印，看起来就像没中断过一样，至此进程热迁移就算完成了
```
# ps -ef | grep piggie
root     30870     1  0 10:51 ?        00:00:00 ./piggie piggie.log
root     30880 29769  0 10:52 pts/0    00:00:00 grep --color=auto piggie
# tailf piggie.log
210
211
212
213
214
215
```
# 容器热迁移
## 准备一个容器
以wordpress为例，首先在源节点node1上创建一个容器
```
# docker run --name mwp -e WORDPRESS_DB_HOST=10.30.252.241:3306 -e WORDPRESS_DB_USER=root -e WORDPRESS_DB_PASSWORD=123456 -p81:80 -d --privileged --security-opt seccomp:unconfined wordpress
ca22b1d7cc1b14ebc3e019d87718c54672b182464734d1b16a6b5f5adaf88b88
```
打开浏览器检查wordpress状态正常

## 创建checkpoint
```
# docker checkpoint create ca22b1d7cc1b c1
```
创建好的image文件默认保存在这个目录下： /var/lib/docker/containers/<container_id>/checkpoints/<checkpoint_name>/

创建checkpoint成功后通过docker ps命令看到容器进程被杀掉了
## 导出源容器的镜像文件
> 注意：导出镜像需要使用save&load，不能使用export&import，原因是存储层的数据是容器运行所需的必备文件，两者的区别参考https://jingsam.github.io/2017/08/26/docker-save-and-docker-export.html

1. 定制镜像

将容器转换为docker镜像
```
# docker commit ca22b1d7cc1b wordpress:v1
```
成功后能看到多出来一个image
```
# docker images
REPOSITORY                              TAG                 IMAGE ID            CREATED             SIZE
wordpress                               v1                  872e4c495369        2 seconds ago       420MB
```

2. 导出镜像

将上一步转换的docker镜像导出为一个输出文件
```
# docker save -o wordpressv1 wordpress:v1
```
在当前目录下能看到导出的镜像文件wordpressv1
```
# ll -h wordpressv1
-rw------- 1 root root 410M Jan 24 12:54 wordpressv1
```
如果不需要了可以删除该镜像文件
```
# docker rmi wordpress:v1
```
## 复制镜像文件到目的节点
将导出的镜像文件复制到目的节点
```
# scp wordpressv1 node2:/home
```

## 恢复
登录到node2节点上
1. 导入镜像
```
# cd /home
# docker load -i wordpressv1
6b237323a705: Loading layer [==================================================>]  7.168kB/7.168kB
Loaded image: wordpress:v1
```
2. 查看导入的image
```
# docker images
REPOSITORY          TAG                 IMAGE ID            CREATED             SIZE
wordpress           v1                  872e4c495369        2 minutes ago       420MB
```
3. 创建容器
```
# docker create --name mwp -e WORDPRESS_DB_HOST=10.30.252.241:3306 -e WORDPRESS_DB_USER=root -e WORDPRESS_DB_PASSWORD=123456 -p81:80 --privileged--security-opt seccomp:unconfined wordpress:v1
00d9e510226fa18f146b09613603e6f85601227eeae62f56191f043cff6effef
```
4. 从node1节点拷贝checkpoint文件到node2节点

scp -r node1:/var/lib/docker/containers/<src_container_id>/checkpoints/<checkpoint_name>/ node2:/var/lib/docker/containers/<dest_container_id>/checkpoints/
```
# scp -r node1:/var/lib/docker/containers/ca22b1d7cc1b14ebc3e019d87718c54672b182464734d1b16a6b5f5adaf88b88/checkpoints/c1 /var/lib/docker/containers/00d9e510226fa18f146b09613603e6f85601227eeae62f56191f043cff6effef/checkpoints
```
5. 恢复启动容器
```
# docker start --checkpoint c1 00d9e510226f
```

## 查看状态
打开浏览器检查WordPress状态正常

> 官网给的例子是在容器内运行一个循环，测试也可以正常迁移
```
1. 启动容器
# docker run -d --name looper --security-opt seccomp:unconfined busybox  \
         /bin/sh -c 'i=0; while true; do echo $i; i=$(expr $i + 1); sleep 1; done'
2. 创建checkpoint
# docker checkpoint create looper c1
3. 检查进程日志
# docker logs looper
4. 转换为镜像
# docker commit 6f71122189b2 busybox:v1
# docker save -o busyboxv1 busybox:v1
5. 拷贝镜像到目的节点
# scp node1:/home/busyboxv1 node2:/home
6. 在目的节点导入镜像
# docker load -i /home/busyboxv1
7.在目的节点创建新容器
# docker create --name looper --security-opt seccomp:unconfined busybox:v1 /bin/sh -c 'i=0; while^Crue; do echo $i; i=$(expr $i + 1); sleep 1; done'
8. 拷贝checkpoint到目的节点
# scp -r node1:/var/lib/docker/containers/6f71122189b2d797ed821e08481e8277289a51458a6090570bad2e57cf25ec45/checkpoints/c1/ node2:/var/lib/docker/containers/22fde8c8571c5655b0ba2377925a5e2791b0858343cfdf0335c3857cc2c2f514/checkpoints/
9. 启动容器
# docker start --checkpoint c1 22fde8c8571c
10. 再次检查进程日志正常，接着上次创建checkpoint的时间点打印
# docker logs 22fde8c8571c
```

# 功能限制
目前docker将criu标记为实验性质的功能，存在以下兼容性问题需要注意
1. TTY，不支持对交互式容器创建checkpoint；
2. Seccomp，如果容器开启了安全计算模式，需要kernel支持（grep CONFIG_SECCOMP=  /boot/config-$(uname -r) 返回CONFIG_SECCOMP=y表示支持）
3. OverlayFS，kernel有个关于OverlayFS的bug在v4.2-rc2版本中被修复
4. Async IO，如果容器使用了AIO需要kernel>=3.19

# 总结
criu能支持热迁移的功能有限，这也是docker没有将它直接封装成跨主机迁移的功能，目前只能在本地实现进程的dump和restore，而且对于在容器内运行的交互式程序不支持，这也是后面使用时要注意的一个问题。
只能说这个功能能解决的问题十分有限，另外还有很多程序不支持热迁移，例如mysql详见https://github.com/checkpoint-restore/criu/issues/484

# 参考资料
1. [criu_docker WIKI](https://criu.org/Docker)
2. [docker-ce安装](https://docs.docker-cn.com/engine/installation/linux/docker-ce/centos/)
3. [docker容器数据迁移](https://blog.csdn.net/weixin_36343850/article/details/80553680)