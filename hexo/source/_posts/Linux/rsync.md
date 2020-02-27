---
layout: post
title:  "rsync 的简单用法"
date:   2018-09-16 23:32:00 +0800
categories: Linux
---
```shell
rsync -av 源 目标
```
例
```shell
rsync -av root@lengqing.org:/root /root/lengqing
```


源目录带/则同步目录下内容，不带/则同步文件夹
例

```shell
rsync -av root@lengqing.org:/root /root/lengqing
```
就会产生/root/lengqing/root目录
而
```shell
rsync -av root@lengqing.org:/root/ /root/lengqing
```
就不会，只会有/root/lengqing目录

