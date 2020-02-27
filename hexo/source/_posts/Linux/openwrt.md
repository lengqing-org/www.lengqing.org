---
layout: post
title:  "新装OpenWRT初次配置"
date:   2019-07-22 21:18:00 +0800
categories: Linux
---
由于新装OpenWRT默认IP是192.168.1.1
然而我内网网段不是192.168.1.0/24，初次访问webUI有点不方便
所以直接在控制台修改IP

```shell
uci set network.lan.ipaddr=192.168.2.6
uci commit
/etc/init.d/network restart
```

然后就可以访问修改后的IP了

然后搜索安装下面的软件包可以获得中文翻译

> luci-i18n-base-zh-cn
