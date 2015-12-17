---
layout: post
title: 如何动态在线扩容root根分区的大小
tags: Ceph
---


```sh
qemu-img resize yourname.img +10G 
```
首先要用命令增加分区大小，针对qemu-kvm使用以上命令

# LVM #

# 非LVM #
```sh
fdisk /dev/sda
d
n
p
1

w
resize2fs /dev/sda1

df -h
```
- 最重要的一步：“删除现在的分区，重新分区”
按d删除现在的分区1，注意：删除后千万不要按w保存！直接按n创建新的分区，然后从原有的柱面开始，一直分到最后的尺寸(默认值两次回车即可，如果之前的分区不是从第一柱面开始，则需要记录之前分区的起始柱面)，新的分区操作完毕后，按w保存。

# URL #
http://www.yunwei123.com/%E9%98%BF%E9%87%8C%E4%BA%91%E7%A3%81%E7%9B%98%E6%89%A9%E5%AE%B9%EF%BC%8D%E9%92%88%E5%AF%B9windowslinux%E6%97%A0%E6%8D%9F%E6%89%A9%E5%AE%B9%E5%88%86%E5%8C%BA%E5%A4%A7%E5%B0%8F%E6%96%B9%E6%B3%95/<br>
http://askubuntu.com/questions/335401/how-to-change-partitions-on-ubuntu-virtual-machine
