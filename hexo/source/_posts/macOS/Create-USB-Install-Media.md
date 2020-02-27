---
layout: post
title:  "制作macOS原版安装启动U盘"
date:   2019-06-06 19:13:00 +0800
categories: macOS
---

### 10.13及以前版本

```shell
sudo /目录/系统安装.app/Contents/Resources/createinstallmedia --volume /Volumes/U盘挂载点 --applicationpath /目录/系统安装.app --nointeraction
```



### 10.14及更新版本

```shell
sudo /目录/系统安装.app/Contents/Resources/createinstallmedia --volume /Volumes/U盘挂载点  --nointeraction
```

与之前的主要区别就是不用再指定系统安装app的位置了。

