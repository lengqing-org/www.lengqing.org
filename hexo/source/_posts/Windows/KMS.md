---
layout: post
title:  "KMS服务器的搭建"
date:   2019-10-17 02:40:42 +0800
categories: Windows
---

### 1. 启动`KMS`服务 端口就不要改了，免得激活时`office`找不到端口

```shell
./vlmcsd-x64-musl-static  -L 0.0.0.0:1688 -Dev
```

### 2. 激活windows , 以管理员身份打开power shell，然执行下列命令, 激活成功会弹窗提示的

```shell
slmgr /skms VPS的IP或者绑定的域名
slmgr /ato
slmgr /xpr
```

### 3. 激活office , 激活成功会弹窗提示的

> 以管理员身份打开命令提示符，进入软件安装目录, 比如说 `office 2016 64位` ，装在`D盘`下
> 安装目录：`D:\Program Files\Microsoft Office\Office16`，执行如下：

```shell
cscript ospp.vbs /sethst:VPS的IP或者绑定的域名
cscript ospp.vbs /act
cscript ospp.vbs /dstatus
```

