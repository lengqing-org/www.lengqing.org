---
layout: post
title:  "新装Proxmox VE之后的第一次开机配置"
date:   2019-04-09 19:26:00 +0800
categories: Linux
---
文章使用版本为5.3-2

#### 改源
Debian源

```shell
nano /etc/apt/sources.list
```

注释或删去原有的源，并添加下面的源

> deb http://mirrors.aliyun.com/debian/ stretch main non-free contrib
>
> deb-src http://mirrors.aliyun.com/debian/ stretch main non-freecontrib 
>
> deb http://mirrors.aliyun.com/debian-security stretch/updates main 
>
> deb-src http://mirrors.aliyun.com/debian-security stretch/updatesmain 
>
> deb http://mirrors.aliyun.com/debian/ stretch-updates mainnon-free contrib 
>
> deb-src http://mirrors.aliyun.com/debian/stretch-updates main non-free contrib 
>
> deb http://mirrors.aliyun.com/debian/ stretch-backports main non-free contrib 
>
> deb-src http://mirrors.aliyun.com/debian/ stretch-backports main non-free contrib

PVE源
```shell
rm -f /etc/apt/sources.list.d/pve-enterprise.list 
echo "deb http://download.proxmox.com/debian/pve stretch pve-no-subscription">/etc/apt/sources.list.d/pve-install-repo.list 
wget http://download.proxmox.com/debian/proxmox-ve-release-5.x.gpg -O /etc/apt/trusted.gpg.d/proxmox-ve-release-5.x.gpg 
```

更新
```shell
apt-get update && apt-get dist-upgrade
```

#### 消除订阅提醒
emmm没看到靠谱简单的方法，不过反正开机第一次登录会提醒，后面就没了，我就不弄了。