---
layout: post
title:  "Ubuntu Server 18.04 修改主机名"
date:   2019-07-18 08:14:00 +0800
categories: Linux
---

```shell
sudo vim /etc/cloud/cloud.cfg
```

找到
>preserve_hostname: false
>
将值修改为true

```shell
sudo hostnamectl set-hostname 新主机名
```

重启
