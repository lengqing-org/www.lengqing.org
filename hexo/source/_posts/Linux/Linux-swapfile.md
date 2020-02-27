---
layout: post
title:  "Linux swapfile 的创建使用"
date:   2018-07-17 01:53:00 +0800
categories: Linux
---

### 以root或sudo运行


#### 创建swapfile
```shell
dd if=/dev/zero of=/swapfile bs=1024 count=2048000
```

#### 启用swapfile
```shell
mkswap /swapfile
swapon /swapfile
```

#### 加入fstab

> /swapfile        swap        swap      defaults         0 0


#### 查看swap
```shell
cat /proc/swaps
```

#### 关闭swap
```shell
swapoff /swapfile
```
#### 删除swapfile
```shell
rm /swapfile
```
