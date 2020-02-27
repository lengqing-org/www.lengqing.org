---
layout: post
title:  "Gentoo Linux的安装"
date:   2019-04-14 22:10:00 +0800
categories: Linux
---
**本文仅为我本人在某次安装Gentoo Linux的时候顺带记录的，并不完整且不完全适用于任何情况。
如欲寻求更为具体的安装过程或者安装教程，请参照官方文档：[Gentoo Wiki 手册](https://wiki.gentoo.org/wiki/Handbook:Main_Page)。**

在[Gentoo Linux下载](https://www.gentoo.org/downloads/)页面选择合适的系统镜像以及stage3包。

### 0 使用LiveCD或LiveDVD引导计算机或虚拟机启动

### 1 登录并正确启用网络

### 2 准备磁盘并正确分区挂载

#### 2.0 使用fdisk分区

fdisk基本用法：
对/dev/sda分区

```shell
fdisk /dev/sda
```

创建GPT分区表
```shell
g
```
创建新分区
```shell
n
```
查看分区
```shell
p
```
更改分区类型
```shell
t
```
保存更改
```shell
w
```

#### 2.1 格式化ESP分区和根目录

```shell
mkfs.fat -F32 /dev/sda1
mkfs.xfs /dev/sda2
```

#### 2.2 挂载根目录和ESP分区

```shell
mount -t xfs /dev/sda2 /mnt/gentoo/
mkdir boot
mount -t vfat /dev/sda1 /mnt/gentoo/boot/
```

### 3 安装stage3

#### 3.0 检查时间并修正

检查当前机器时间（建议使用UTC时间）
```shell
date
```
如不正确，使用ntp命令同步网络时间
```shell
ntpd -q -g
```
#### 3.1 选择一个stage包下载并检查包完整性

选择自己想要的stage包，我选的是AMD64架构的默认包，也是不带标记的包，默认为multilib非systemd的版本。
下载下来并复制到挂载的根目录，我直接下载到NAS中，并通过局域网wget到挂载目录的。
检查哈希值并与官网上的对比。

#### 3.2 解压stage3

这里需要将stage3归档包的文件名更改，但参数要加的。
```shell
tar xpvf stage3.tar.xz --xattrs-include='*.*' --numeric-owner
```
#### 3.3 配置编译选项
```shell
nano -w /mnt/gentoo/etc/portage/make.conf
```
在文本末行添加下面的文本，其中12可以替换成你的CPU的线程数，也可以设为线程数+1

> MAKEOPTS="-j12"

其它内容没有修改的必要，保存退出nano

### 4 安装基础系统

#### 4.0 选择镜像站点

我这里选择清华tuna开源镜像站

    nano -w /mnt/gentoo/etc/portage/make.conf

在最后一行加入以下文本

> GENTOO_MIRRORS="https://mirrors.tuna.tsinghua.edu.cn/gentoo"

#### 4.1 Gentoo ebuild 软件仓库
```shell
mkdir --parents /mnt/gentoo/etc/portage/repos.conf
cp /mnt/gentoo/usr/share/portage/config/repos.conf /mnt/gentoo/etc/portage/repos.conf/gentoo.conf
```

这里用的是官方的仓库，速度较慢，还是换成tuna的吧
替换sync-uri开头的一行
> sync-uri = rsync://mirrors.tuna.tsinghua.edu.cn/gentoo-portage

#### 4.2 复制DNS信息
```shell
cp --dereference /etc/resolv.conf /mnt/gentoo/etc/
```

#### 4.3 挂载必要的文件系统
```shell
mount --types proc /proc /mnt/gentoo/proc
mount --rbind /sys /mnt/gentoo/sys
mount --make-rslave /mnt/gentoo/sys
mount --rbind /dev /mnt/gentoo/dev
mount --make-rslave /mnt/gentoo/dev
```

#### 4.4 进入新环境
```shell
chroot /mnt/gentoo /bin/bash
source /etc/profile
export PS1="(chroot) ${PS1}"
```

#### 4.5 配置Portage

从网站安装ebuild 数据库快照
```shell
emerge-webrsync
```

更新Portage ebuild 数据库
```shell
emerge --sync
```

如果有更新提示，可以选择按提示更新，但完全没有必要


#### 4.6 选择正确的配置文件
```shell
eselect profile list
```

会出现下面的内容
>(chroot) livecd / # eselect profile list
>
>Available profile symlink targets:
>
>[1]   default/linux/amd64/13.0 (stable)
>
>[2]   default/linux/amd64/13.0/selinux (dev)
>
>[3]   default/linux/amd64/13.0/desktop (stable)
>
>[4]   default/linux/amd64/13.0/desktop/gnome (stable)
>
>[5]   default/linux/amd64/13.0/desktop/gnome/systemd (stable)
>
>[6]   default/linux/amd64/13.0/desktop/plasma (stable)
>
>[7]   default/linux/amd64/13.0/desktop/plasma/systemd (stable)
>
>[8]   default/linux/amd64/13.0/developer (stable)
>
>[9]   default/linux/amd64/13.0/no-multilib (stable)
>
>[10]  default/linux/amd64/13.0/systemd (stable)
>
>[11]  default/linux/amd64/13.0/x32 (dev)
>
>[12]  default/linux/amd64/17.0 (stable) *
>
>[13]  default/linux/amd64/17.0/selinux (stable)
>
>[14]  default/linux/amd64/17.0/hardened (stable)
>
>[15]  default/linux/amd64/17.0/hardened/selinux (stable)
>
>[16]  default/linux/amd64/17.0/desktop (stable)
>
>[17]  default/linux/amd64/17.0/desktop/gnome (stable)
>
>[18]  default/linux/amd64/17.0/desktop/gnome/systemd (stable)
>
>[19]  default/linux/amd64/17.0/desktop/plasma (stable)
>
>[20]  default/linux/amd64/17.0/desktop/plasma/systemd (stable)
>
>[21]  default/linux/amd64/17.0/developer (stable)
>
>[22]  default/linux/amd64/17.0/no-multilib (stable)
>
>[23]  default/linux/amd64/17.0/no-multilib/hardened (stable)
>
>[24]  default/linux/amd64/17.0/no-multilib/hardened/selinux (stable)
>
>[25]  default/linux/amd64/17.0/systemd (stable)
>
>[26]  default/linux/amd64/17.0/x32 (dev)
>
>[27]  default/linux/amd64/17.1 (dev)
>
>[28]  default/linux/amd64/17.1/selinux (dev)
>
>[29]  default/linux/amd64/17.1/hardened (dev)
>
>[30]  default/linux/amd64/17.1/hardened/selinux (dev)
>
>[31]  default/linux/amd64/17.1/desktop (dev)
>
>[32]  default/linux/amd64/17.1/desktop/gnome (dev)
>
>[33]  default/linux/amd64/17.1/desktop/gnome/systemd (dev)
>
>[34]  default/linux/amd64/17.1/desktop/plasma (dev)
>
>[35]  default/linux/amd64/17.1/desktop/plasma/systemd (dev)
>
>[36]  default/linux/amd64/17.1/developer (dev)
>
>[37]  default/linux/amd64/17.1/no-multilib (dev)
>
>[38]  default/linux/amd64/17.1/no-multilib/hardened (dev)
>
>[39]  default/linux/amd64/17.1/no-multilib/hardened/selinux (dev)
>
>[40]  default/linux/amd64/17.1/systemd (dev)
>
>[41]  default/linux/amd64/17.0/musl (exp)
>
>[42]  default/linux/amd64/17.0/musl/hardened (exp)
>
>[43]  default/linux/amd64/17.0/musl/hardened/selinux (exp)
>
>[44]  default/linux/amd64/17.0/uclibc (exp)
>
>[45]  default/linux/amd64/17.0/uclibc/hardened (exp)
>

其中标*的为当前选择的配置，当前选择的是12
如果需要修改为其它的配置，运行下面的命令，将其中的12改为所需要的配置代号
```shell
eselect profile set 12
```

#### 4.7 更新@world集合
```shell
emerge --ask --verbose --update --deep --newuse @world
```
需要输入Yes确认

#### 4.8 配置USE变量（这是Gentoo的重要特色，但需要花时间深入了解）
这一项旨在添加编译参数，在没有深入了解的时候可以忽略

#### 4.9 配置 ACCEPT_LICENSE 变量
这一项旨在选择接受的许可证，不同的许可证可安装的软件包有不同，这里没有特别需求，默认就可以

#### 4.10 配置时区
查看可用时区
```shell
ls /usr/share/zoneinfo
```
我这里选择 亚洲/上海
```shell
echo "Asia/Shanghai" > /etc/timezone
```

重新配置sys-libs/timezone-data包
```shell
emerge --config sys-libs/timezone-data
```

#### 4.11 配置地区
**（这里需要注意的是，不支持中文的终端或者没有安装中文字体，都将导致中文乱码，因此不建议在无桌面环境下设置默认地区为中文地区）**
打开配置文件
```shell
nano -w /etc/locale.gen
```

在末尾添加
>en_US ISO-8859-1
>
>en_US.UTF-8 UTF-8
>
>zh_CN GBK 
>
>zh_CN.UTF-8 UTF-8

运行locale-gen生成/etc/locale.gen文件中指定的所有地区
```shell
locale-gen
```

设定系统级别的区域设置
```shell
eselect locale list
```
会出现以下内容
>(chroot) livecd / # eselect locale list
>
>Available targets for the LANG variable:
>
>[1]   C
>
>[2]   C.utf8
>
>[3]   en_US
>
>[4]   en_US.iso88591
>
>[5]   en_US.utf8
>
>[6]   POSIX
>
>[7]   zh_CN
>
>[8]   zh_CN.gbk
>
>[9]   zh_CN.utf8
>
>[ ]   (free form)

这里选择区域设置9
```shell
eselect locale set 9
```

为避免因为缺少中文字体而导致中文乱码的情况，先安装三个文泉驿字体
```shell
emerge media-fonts/wqy-bitmapfont
emerge media-fonts/wqy-zenhei
emerge media-fonts/wqy-microhei
```

现在重新加载环境
```shell
env-update && source /etc/profile && export PS1="(chroot) $PS1"
```

**到此为止可以删除并清理不需要的文件，打包自己配置的基础系统文件以供下次安装可以免去配置的过程**

### 5 配置内核

#### 5.0 安装内核源码
```shell
emerge --ask sys-kernel/gentoo-sources
```

查看源码
```shell
ls -l /usr/src/linux
```

#### 5.1 手动编译（建议，但生手还是自动编译吧，然而自动编译也有开不了机的可能）
使用
```shell
lspci
```

查看pci设备

默认没有安装pci工具包，需要安装才能使用
```shell
emerge --ask sys-apps/pciutils
```

进入源码目录
```shell
cd /usr/src/linux
```

开始配置
```shell
make menuconfig
```

配置省略

编译与安装模块
```shell
make && make modules_install
```

安装内核
```shell
make install
```

#### 5.2 生成一个initramfs
需要用到genkernel包，需要安装
```shell
emerge --ask sys-kernel/genkernel
```

如有遇到USE change警告，使用
```shell
etc-update
```

并选择-3或-5来更新USE配置

生成initramfs
```shell
genkernel --install initramfs
```

为了在initramfs中启用特定的支持，比如LVM或RAID，需要加入特定的参数，比如
```shell
genkernel --lvm --mdadm --install initramfs
```

查看生成的文件
```shell
ls /boot/initramfs*
```

#### 5.3 使用genkernel自动编译

同样需要使用genkernel包，没安装的需要装一下
```shell
emerge --ask sys-kernel/genkernel
```

如有遇到USE change警告，使用
```shell
etc-update
```

并选择-3或-5来更新USE配置

编辑fstab文件来使/boot/指向到正确的设备（根目录/在这里不做要求，但在后面的配置系统里有要求）
```shell
nano -w /etc/fstab
```

确保所有行都被#开头注释
添加
>UUID=XXXXX   /       xfs    noatime         0 1
>UUID=XXX     /boot   vfat   noauto,noatime  1 2

其中XXX的部分需要自己查自己的分区UUID，通过下面的命令可以看到
```shell
ls -l /dev/disk/by-uuid/
```

自动编译安装
```shell
genkernel all
```

查看安装好的内核与生成的initramfs
```shell
ls /boot/kernel* /boot/initramfs*
```

#### 5.4 内核模块
在/etc/modules-load.d/*.conf列出需要自动加载的模块，每行一个模块。
如有必要，应在/etc/modprobe.d/*.conf文件中设置模块的额外选项。

查看所有可用模块，其中<内核版本>需要替换为刚刚编译的版本
```shell
find /lib/modules/<内核版本>/ -type f -iname '*.o' -or -iname '*.ko' | less
```

如有需要加载的模块，在/etc/modules-load.d/中创建*.conf文件，文件名不重要，可根据设备与自己的喜好命名
```shell
mkdir -p /etc/modules-load.d
```

如3Com网卡设备，要自动加载3c59x.ko模块
```shell
nano -w /etc/modules-load.d/network.conf
```

向其中加入
>3c59x

保存即可

#### 5.5 安装固件
```shell
emerge --ask sys-kernel/linux-firmware
```

### 6 配置系统

#### 6.0 文件系统信息

关于fstab文件
在编译内核部分没有修改，或只修改了/boot/部分的，这里需要做修改
```shell
nano -w /etc/fstab
```

确保所有行都被#开头注释
添加
>UUID=XXXXX   /       xfs    noatime         0 1
>UUID=XXX     /boot   vfat   noauto,noatime  1 2

其中XXX的部分需要自己查自己的分区UUID，通过下面的命令可以看到
```shell
ls -l /dev/disk/by-uuid/
```

#### 6.1 网络信息

主机名、域名信息
```shell
nano -w /etc/conf.d/hostname
```

设置主机名变量，选择主机名
>hostname="gentoo"

如果你需要一个域名，在/etc/conf.d/net中设定
/etc/conf.d/net文件默认不存在，因此需要创建
```shell
nano -w /etc/conf.d/net
```

设定dns_domain的变量值为你的域名
>dns_domain_lo="homenetwork"

如果你有一个NIS域（如果你不知道这是什么，就说明你没有），你也需要定义一个
设定nis_domain的变量值为你的NIS域名
>nis_domain_lo="my-nisdomain"

配置网络
首先安装net-misc/netifrc
```shell
emerge --ask --noreplace net-misc/netifrc
```
系统默认使用DHCP。如果使用DHCP的话，你需要安装一个DHCP客户端
如果你需要配置你的网络连接，打开 /etc/conf.d/net
```shell
nano -w /etc/conf.d/net
```

设置 config_eth0 和 routes_eth0 输入IP地址信息和路由信息
>config_eth0="192.168.2.2 netmask 255.255.255.0 brd 192.168.2.255"
>routes_eth0="default via 192.168.2.1"

要使用DHCP
>config_eth0="dhcp"

**所有的eth0需要改为你自己的网络接口名称，可能是enp2s1、ens0或者其它
这里需要注意的是，重启之后的网络跟Chroot环境下的可能不一样，需要重启之后重新修正
假如接口名称不正确，系统将不能正确地自动联网**

在启动时自动启用网络链接（这里的eth0依旧需要改成你自己的网络接口名称）
```shell
cd /etc/init.d
ln -s net.lo net.eth0
rc-update add net.eth0 default
```

hosts 文件
```shell
nano -w /etc/hosts
```

定义的是现在系统
>127.0.0.1     gentoo.homenetwork gentoo localhost

定义你网络上的其它系统
>192.168.0.5   jenny.homenetwork jenny
>192.168.0.6   benny.homenetwork benny

#### 6.2 启用PCMCIA
```shell
emerge --ask sys-apps/pcmciautils
```

#### 6.3 系统信息
Root 密码
```shell
passwd
```

配置引导和启动
使用/etc/rc.conf配置系统的服务，启动和关闭
```shell
nano -w /etc/rc.conf
```

打开/etc/conf.d/keymaps 来处理键盘设置，默认美式英文键盘
```shell
nano -w /etc/conf.d/keymaps
```

机器时间，默认是UTC时间
```shell
nano -w /etc/conf.d/hwclock
```

如果你机器上的时钟不用UTC
>clock="UTC"

需要改成
>clock="local"

否则，你的时钟就有可能出现偏差

### 7 安装系统工具
#### 7.0 系统日志工具
```shell
emerge --ask app-admin/sysklogd
rc-update add sysklogd default
```

#### 7.1 Cron守护进程
```shell
emerge --ask sys-process/cronie
rc-update add cronie default
```

#### 7.2 文件索引
```shell
emerge --ask sys-apps/mlocate
```

#### 7.3 远程访问
```shell
rc-update add sshd default
```

如果需要终端访问，请在 /etc/inittab中取消注释控制台部分
```shell
nano -w /etc/inittab
```

找到# SERIAL CONSOLES一行，下面两行取消注释
>s0:12345:respawn:/sbin/agetty 9600 ttyS0 vt100
>s1:12345:respawn:/sbin/agetty 9600 ttyS1 vt100

#### 7.4 文件系统工具

|     文件系统      |        软件包        |
| :---------------: | :------------------: |
|    Ext2, 3, 4     |   sys-fs/e2fsprogs   |
|        XFS        |   sys-fs/xfsprogs    |
|     ReiserFS      | sys-fs/reiserfsprogs |
|        JFS        |   sys-fs/jfsutils    |
| VFAT (FAT32, ...) |  sys-fs/dosfstools   |
|       Btrfs       |  sys-fs/btrfs-progs  |




#### 7.5 网络工具
安装DHCP客户端
```shell
emerge --ask net-misc/dhcpcd
```

可选：安装PPPoE客户端
```shell
emerge --ask net-dialup/ppp
```

可选：安装无线网络工具
```shell
emerge --ask net-wireless/iw net-wireless/wpa_supplicant
```

### 8 配置引导加载程序

#### 8.0 选择引导器（本篇文章仅讲述grub2在UEFI下的应用）
安装GRUB2（UEFI必须先在make.conf加入GRUB_PLATFORMS="efi-64"）
```shell
echo 'GRUB_PLATFORMS="efi-64"' >> /etc/portage/make.conf
emerge --ask --verbose sys-boot/grub:2
```

激活GRUB2
```shell
grub-install --target=x86_64-efi --efi-directory=/boot
```

在只支持EFI系统分区（ESP）中.EFI文件的 /efi/boot/目录中，需要添加--removable参数
```shell
grub-install --target=x86_64-efi --efi-directory=/boot --removable
```

这将创建UEFI规范定义的默认目录，然后将 grubx64.efi 文件复制到由同一规范定义的“默认”EFI文件位置

配置GRUB2
```shell
grub-mkconfig -o /boot/grub/grub.cfg
```

### 9 收尾安装工作（这一步也可以在重启进入新系统登录root之后再做）

#### 9.0 用户管理
添加一个日常使用的用户
```shell
useradd -m -G users,wheel,audio -s /bin/bash 用户名
```

#### 修改用户密码
```shell
passwd 用户名
```

#### 9.1 磁盘清理
```shell
rm /stage3*
```

#### 安装到此结束
