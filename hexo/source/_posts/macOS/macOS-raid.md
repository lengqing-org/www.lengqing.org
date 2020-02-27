---
layout: post
title:  "macOS下的磁盘阵列"
date:   2019-06-09 23:45:00 +0800
categories: macOS
---

## macOS自带的工具：RAID助理
1 首先打开磁盘工具，左上角的菜单栏 --> 文件 --> RAID助理

![RAID助理](https://blog.lengqing.org/pics/blog/macos_raid/1.png)   

2 选择需要的RAID类型，这里所提供的极为有限，并且不能组合（先RAID0后RAID1之类的）

![choose-raid](https://blog.lengqing.org/pics/blog/macos_raid/2.png) 

3 选择所要加入的磁盘

![choose-disk1](https://blog.lengqing.org/pics/blog/macos_raid/3.png) 

![choose-disk2](https://blog.lengqing.org/pics/blog/macos_raid/4.png)  

4 设置RAID属性（仅能设置命名以及分区格式，块大小）

![configure1](https://blog.lengqing.org/pics/blog/macos_raid/5.png) 

![configure2](https://blog.lengqing.org/pics/blog/macos_raid/6.png) 

5 完成

![done1](https://blog.lengqing.org/pics/blog/macos_raid/7.png) 

![done2](https://blog.lengqing.org/pics/blog/macos_raid/8.png)  

6 完成后变可以在磁盘工具中看到

![done3](https://blog.lengqing.org/pics/blog/macos_raid/9.png)

## 开源的openZFS

1 首先从[Open-ZFS官网](https://openzfsonosx.org/)下载macOS版，安装预编译版或者使用源码安装
我这里采用预编译安装，下载到的dmg镜像打开后选择对应系统版本的pkg安装器双击安装

![安装](https://blog.lengqing.org/pics/blog/macos_raid/10.png)

2 打开磁盘工具或者系统报告，或者直接终端查看需要添加入ZFS存储池的磁盘BSD名称，我这里4块盘分别是disk0,disk1,disk3,disk4

3 使用命令行创建存储池

```shell
sudo zpool create  zfsrz1_test raidz1 disk0 disk1 disk3 disk4
```

输入账户密码，回车确认即可
此时已经能在磁盘工具中看到创建好的存储池（虚拟磁盘）

![DiskAssistant](https://blog.lengqing.org/pics/blog/macos_raid/11.png)

访达（Finder）中也能够看到

![Finder](https://blog.lengqing.org/pics/blog/macos_raid/12.png) 

**需要注意的是，不知何故，往这个磁盘中写入文件需要输入密码验证**

4 使用命令行维护存储池
我只是做个测试，想要维护的可以自己查找文档

5 销毁（删除）存储池

```shell
sudo zpool destroy zfsrz1_test 
```




## 硬RAID卡以及其它商业软件

硬RAID卡因为目前Mac机器拓展性差，并且可能存在兼容性问题，也因为我没钱，暂时不考虑

其它商业软件因为我没钱，暂时不考虑