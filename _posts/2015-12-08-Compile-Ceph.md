---
layout: post
title: Ceph Compile
tags: Ceph
---

Ceph源码编译问题记录
# 代码下载 #


1. 从官网下载源码
链接：http://download.ceph.com/tarballs/ceph-9.2.0.tar.gz
2. 从github克隆代码
注意：从 https://github.com/ceph/ceph.git 上克隆下来的9.2.0代码有问题。
我遇到的问题是个语法错误：
<br>
make 时报错：
./include/rados/memory.h:1:1: error: expected unqualified-id before ‘.’ token
<br>
如果从官网上直接下载压缩包是没问题的，但是压缩包内缺少一个文件夹debian，这个文件夹在安装依赖包的时候有用，还有打包发布也有用，所以我们要把上面两个源码包都下载下来然后手动拷贝ceph/src/debian文件夹到ceph-9.2.0相同目录下面。


# 如何编译 #
```sh
tar xvf ceph-9.2.0.tar.gz
cd ceph-9.2.0
./install-deps.sh
./configure
make -j4
make install
```

# 问题1 #
A compiler with support for C++11 language features is required. 
重新安装高版本的gcc，且版本必须要 > 4.6 以上
gcc版本必须要 >4.6以上，由于我的系统安装了两个版本的gcc(4.3和4.8)，所以我需要删除旧版本的快捷方式，重新建立gcc g++的路径
```sh
rm /usr/bin/cc
rm /usr/bin/gcc
rm /usr/bin/c++
rm /usr/bin/g++
ln -s /usr/bin/gcc-4.8 /usr/bin/gcc
ln -s /usr/bin/gcc-4.8 /usr/bin/cc
ln -s /usr/bin/g++-4.8 /usr/bin/g++
ln -s /usr/bin/g++-4.8 /usr/bin/c++

apt-get install gcc g++
```

# 问题2 #
checking for snappy_compress in -lsnappy... no
```sh
zypper ar http://download.opensuse.org/repositories/devel:/languages:/erlang/SLE_11_SP4/devel:languages:erlang.repo
zypper in snappy snappy-devel
```

# 问题3 #
```sh
./configure --without-cryptopp --without-tcmalloc --without-libxfs
configure: error: libleveldb not found
```
```sh
#install leveldb
wget https://leveldb.googlecode.com/files/leveldb-1.14.0.tar.gz
tar zxvf leveldb-1.14.0.tar.gz
cd leveldb-1.14.0
make
cp libleveldb.* /usr/lib
cp -r include/leveldb /usr/local/include/
```

# 问题4 #
no suitable crypto library found

解决方法
apt-get install libnss3-1d-dev

zypper in mozilla-nss-devel

# 问题5 #
```sh
make 
报错：
./include/rados/memory.h:1:1: error: expected unqualified-id before ‘.’ token
 ../memory.h
```
vi src/include/rados/memory.h
vi src/include/rados/buffer.h
将 ../memory.h 改成 #include "../memory.h"
将 ../buffer.h 改成 #include "../buffer.h"

# 问题6 #
failed to run libtoolize: No such file or directory
```sh
apt-get install libtool
```

# 问题7 #
autogen.sh > aclocal not found)
```sh
apt-get install automake
```


# 问题8 #
configure: error: libsnappy not found
```sh
apt-get install libsnappy-dev
```

# 问题9 #
configure: error: libleveldb not found
```sh
apt-get install libleveldb-dev
```

# 问题10 #
configure: error: blkid/blkid.h not found (libblkid-dev, libblkid-devel)
```sh
apt-get install libblkid-dev
```

# 问题11 #
configure: error: libudev.h not found (libudev-dev, libudev-devel)
```sh
apt-get install libudev-dev
```

# 问题12 #
configure: error: libkeyutils not found (libkeyutils-dev, keyutils-libs-devel)
```sh
apt-get install libkeyutils-dev
```

# 问题13 #
configure: error: no FUSE found (use --without-fuse to disable)
```sh
apt-get install fuse libfuse-dev
```

# 问题14 #
configure: error: no tcmalloc found (use --without-tcmalloc to disable)
```sh
apt-get install google-perftools libgoogle-perftools-dev 
```

# 问题15 #
configure: error: no libatomic-ops found (use --without-libatomic-ops to disable)
```sh
apt-get install libatomic-ops-dev 
```

# 问题16 #
configure: error: libaio not found
```sh
apt-get install libaio-dev
```

# 问题17 #
configure: error: xfs/xfs.h not found (--without-libxfs to disable)
```sh
apt-get install xfslibs-dev
```

# 问题18 #
/usr/bin/ld: cannot find -ledit
```sh
apt-get install libedit-dev
```

# 问题17 #
No matching distribution found for setuptools
```sh
apt-get install python-pip
```

# 问题19 #
root@ubuntu:/work/ceph-9.2.0# ceph -s
OSError: librados.so.2: cannot open shared object file: No such file or directory
```sh
ldconfig
```

# 运行ceph服务端 #
可以执行以下命令部署一个开发模式的ceph集群
```sh
cd src
install -d -m0755 out dev/osd0
./vstart.sh -n -x -l
# check that it's there
./ceph health
```




