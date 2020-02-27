---
layout: post
title:  "Arch Linux 的安装"
date:   2018-10-01 18:19:50 +0800
categories: Linux
---

**本文参照 Arch Linux 官方 Wiki ，地址：[Installation guide ](https://wiki.archlinux.org/index.php/Installation_guide_(%E7%AE%80%E4%BD%93%E4%B8%AD%E6%96%87)#)**
**现在已经提供了与本文环境相搭配的安装脚本，在[https://github.com/lengqing-org/arch-install](https://github.com/lengqing-org/arch-install)里可以根据README使用。**

##  创建虚拟机（这不是必须的，也可以不一样，实机建议UEFI引导，本文只介绍UEFI引导）
主要就是客户机系统类型的选择，
![system](https://blog.lengqing.org/pics/blog/arch_install/1.png)

还有就是引导的选择，我这里选用UEFI引导。
![UEFI](https://blog.lengqing.org/pics/blog/arch_install/2.png)

##  准备启动盘
下载启动镜像，由于在中国大陆地区，我选择网易163的下载地址：
[http://mirrors.163.com/archlinux/iso/](http://mirrors.163.com/archlinux/iso/)
下载好后检查 MD5 值，
然后可以自行制作启动U盘，或者挂载进虚拟机。

##  开机
![start](https://blog.lengqing.org/pics/blog/arch_install/3.png)
准备好了
![already](https://blog.lengqing.org/pics/blog/arch_install/4.png)

##  联网以及更新系统时间

```shell
dhcpcd
ping -c 4 baidu.com
```

查看结果显示可以 ping 通，联网成功
![network](https://blog.lengqing.org/pics/blog/arch_install/5.png)

更新系统时间，没啥输出，不用管

```shell
timedatectl set-ntp true
```

##  分区及挂载分区
列出磁盘

```shell
fdisk -l
```

![disklist](https://blog.lengqing.org/pics/blog/arch_install/6.png)
我这里第一块是 60GiB 的盘，设备是 /dev/sda 。
第二块应该是启动盘，不用管。

对 /dev/sda 进行分区
```shell
fdisk /dev/sda
```

创建 GPT 分区表
```shell
g
```
创建一个新的分区
```shell
n
```
输入分区号，从1开始
```shell
1
```
输入起始扇区，默认回车即可
回车
输入 +200M 创建一个 esp 分区
```shell
+200M
```
继续创建新分区做根目录分区
```shell
n
```
回车
回车
由于没有其它分区，硬盘剩余空间都分配，所以直接回车分配所有空间
回车

查看创建的分区
```shell
p
```
可以看到两个分区，分别是 200M 的 sda1 和 59.8G 的 sda2 （都没有格式化）
![diskparts](https://blog.lengqing.org/pics/blog/arch_install/7.png)

改变 200M 的esp 分区的类型
```shell
t
```
选择分区号
```shell
1
```
选择分区类型，按L可以列出所有类型，应该序号1是"EFI System"
```shell
1
```
![esp](https://blog.lengqing.org/pics/blog/arch_install/8.png)
保存更改
```shell
w
```
格式化 esp 分区和根分区
```shell
mkfs.fat -F32 /dev/sda1
mkfs.ext4 /dev/sda2
```

挂载根分区
```shell
mount /dev/sda2 /mnt
```
挂载 esp 分区
```shell
mkdir /mnt/boot
mount /dev/sda1 /mnt/boot
```

## 安装基本系统
选择镜像源
编辑源列表文件
```shell
nano /etc/pacman.d/mirrorlist
```
将 China 的那一行挪到最前面（也可以直接在最前面加上 China 的源地址，上面的具有优先权）
找到那一行，光标在最前面的时候直接按下 Ctrl+K ，然后回到第一行按下 Ctrl+U ，手打也行
![mirror](https://blog.lengqing.org/pics/blog/arch_install/9.png)
然后按下 Ctrl+X ，输入 y ，回车，即可保存

安装基本包（ base-devel 不是必须的，可以不加）
```shell
pacstrap /mnt base linux linux-firmware
```
![base](https://blog.lengqing.org/pics/blog/arch_install/10.png)
这个样子就是系统装好了，只不过还不能开机


##  开机前配置
生成自动挂载分区的 fstab 文件
```shell
genfstab -U /mnt >> /mnt/etc/fstab
```
检查是否正确
```shell
cat /mnt/etc/fstab
```
![fstab](https://blog.lengqing.org/pics/blog/arch_install/11.png)
理论上格式应该与我的完全相同，假如格式差别很大的可能就有问题了，也许手动检查 UUID 并创建 fstab 文件是个好主意。

切换到新系统操作
```shell
arch-chroot /mnt
```
![chroot](https://blog.lengqing.org/pics/blog/arch_install/12.png)
如图所示，即为成功

设置时区（我比较靠近上海，便以上海为例）
```shell
ln -sf /usr/share/zoneinfo/Asia/Shanghai /etc/localtime
```
根据硬件时间调整时间（ UTC 时间为例）
```shell
hwclock --systohc
```

本地化
编辑文件
```shell
nano /etc/locale.gen
```
将下面的几行前面的 # 删掉，保存
> en_US.UTF-8 UTF-8
>
> zh_CN.UTF-8 UTF-8
>
> zh_TW.UTF-8 UTF-8

接着执行 locale-gen 以生成 locale 讯息：
```shell
locale-gen
```

选择英文作为默认语言（暂不推荐中文，以防乱码，等装了字体桌面什么的可以考虑中文）
```shell
nano /etc/locale.conf
```
加入以下内容并保存
> LANG=en_US.UTF-8

设置主机名
```shell
nano /etc/hostname
```
输入你要的主机名并保存

编辑 hosts 文件
加入下面的内容，其中的 arch 替换为你的主机名
> 127.0.0.1   localhost.localdomain   localhost
>
> ::1     localhost.localdomain   localhost
>
> 127.0.1.1   arch.localdomain  arch

设置 root 密码
```shell
passwd
```

Intel CPU 需要安装的 intel-ucode
```shell
pacman -S intel-ucode
```

安装引导
```shell
pacman -S os-prober grub efibootmgr
```

部署grub
```shell
grub-install --target=x86_64-efi --efi-directory=/boot --bootloader-id=grub
```

生成配置文件（我这里会出错）
```shell
grub-mkconfig -o /boot/grub/grub.cfg
```

报错
> warning failed to connect to lvmetad，falling back to device scanning.

编辑 /etc/lvm/lvm.conf 文件，找到 use_lvmetad = 1 将1修改为0
（这一行在很下面，翻半天），保存，重新生成配置文件


##  重启进入新系统
安装基本上结束了
```shell
exit
reboot
```
顺便拔掉启动盘
开机就算是进入新系统了，如果不能开机我也不管了，教程到这里结束
正常的开机应该跟我的一样
![boot](https://blog.lengqing.org/pics/blog/arch_install/13.png)
然后滚更新，最新的，很棒！
![update](https://blog.lengqing.org/pics/blog/arch_install/14.png)


