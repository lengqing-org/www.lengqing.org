---
layout: post
title:  "ESXi直通硬盘"
date:   2019-07-22 21:12:00 +0800
categories: Linux
---
参考文章：[本地存储的裸设备映射](https://kb.vmware.com/s/article/1017530?lang=zh_CN)

1 ssh连接esxi

2 列出磁盘，也可以在web控制台中查看

```shell
ls -l /vmfs/devices/disks
```

3 将设备配置为 RDM，并将 RDM 指针文件输出到您所选的目标
```shell
vmkfstools -z /vmfs/devices/disks/磁盘设备名 /vmfs/volumes/存储池/目录/虚拟磁盘名.vmdk
```

4 在虚拟机中添加映射的虚拟磁盘

