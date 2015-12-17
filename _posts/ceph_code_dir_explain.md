---
layout: post
title: Ceph源代码目录结构详解
tags: Ceph
---

Ceph源代码目录结构详解
---
从GitHub上Clone的Ceph项目，其目录下主要文件夹和文件的内容为：

*   [根目录](#root_dir)
*   [src目录](#src_dir)

<h2 id="root_dir">根目录</h2>
* [src]：各功能某块的源代码
* [qa]：各个模块的功能测试（测试脚本和测试代码）
* [wireshark]：#wireshark的ceph插件。
* [admin]：管理工具，用于架设文档服务器等
* [debian]：用于制作debian（Ubuntu）安装包的相关脚本和文件
* [doc]：用于生成项目文档，生成结果参考http://ceph.com/docs/master/
* [man]：ceph各命令行工具的man文件
* configure.ac：用于生成configure的脚本
* Makefile.am：用于生成Makefile的脚本
* autogen.sh：负责生成configure。
* do_autogen.sh：生成configure的脚本，实际上通过调用autogen.sh实现
* ceph.spec.in：RPM包制作文件

<h2 id="src_dir">src目录</h2>

* [include]：头文件，包含各种基本类型的定义，简单通用功能等。
* [common]：共有模块，包含各类共有机制的实现，例如线程池、管理端口、节流阀等。
* [log]：日志模块，主要负责记录本地log信息（默认/var/log/ceph/目录）
* [global]：全局模块，主要是声明和初始化各类全局变量（全局上下文）、构建驻留进程、信号处理等。
* [auth]：授权模块，实现了三方认知机制。
* [crush]：Crush模块，Ceph的数据分布算法
* [msg]：消息通讯模块，包括用于定义通讯功能的抽象类Messenger以及目前的实现SimpleMessager
* [messages]：消息模块，定义了Ceph各节点之间消息通讯中用到的消息类型。
* [os]：对象（Object Store）模块，用于实现本地的对象存储功能，
* [osdc]：OSD客户端（OSD Client），封装了各类访问OSD的方法。
* [mon]：mon模块
* [osd]：osd部分
* [mds]：mds模块
* [rgw]：rgw模块的
* [librados]：rados库模块的代码
* [librdb]：libbd库模块的代码
* [client]：client模块，实现了用户态的CephFS客户端
* [mount]：mount模块
* [tools]：各类工具
* [test]：单元测试
* [perfglue]：与性能优化相关的源代码
* [json_spirit]：外部项目json_spirit
* [leveldb]：外部项目leveldb from google
* [gtest]：gtest单元测试框架
* [doc]：关于代码的一些说明文档
* [bash_completion]：部分bash脚本的实现
* [pybind]：python的包装器
* [script]：各种python脚本
* [upstart]：各种配置文件
