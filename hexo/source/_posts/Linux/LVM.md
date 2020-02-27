---
layout: post
title:  "LVM的基本使用方法"
date:   2019-12-02 00:17:00 +0800
updated: 2019-12-02 21:36:00 +0800
categories: Linux
---


# LVM的基本使用方法

LVM是Logical Volume Manager，逻辑卷管理器，逻辑卷类似Windows下的跨区卷吧。

原先我一直觉得LVM用处有限，自己没用上，一直没用，也就一直没写怎么用，后来发现还是挺有用的，尤其是搭配RAID来使用。

有几个概念先了解一下：

|     名称     |      英文       | 缩写 |                    解释                     |
| :----------: | :-------------: | :--: | :-----------------------------------------: |
| 物理存储介质 |        -        |  -   |   在系统里一般是硬盘，也可以是块存储设备    |
|    物理卷    | PhysicalVolume  |  PV  |             磁盘的分区转化来的              |
|     卷组     |   VolumeGroup   |  VG  | 物理卷组成的组，在LVM里等同于虚拟磁盘的意思 |
|    逻辑卷    |  LogicalVolume  |  LV  | 建立于卷组之上，在LVM里等同于虚拟分区的意思 |
|      -       | PhysicalExtents |  PE  |                      -                      |
|      -       | LogicalExtents  |  LE  |                      -                      |

LVM大致上的概念就是把一个或多个物理磁盘的一个或多个分区合并在一起，称作卷组，也可以理解为变成了一块更大的虚拟磁盘。

但是物理磁盘的分区不能直接合并到卷组里，需要转化，而转化之后的分区就称为物理卷。

然后再创建逻辑卷，就相当于在这块更大的虚拟磁盘上划分分区。

然后逻辑卷就可以格式化，挂载等基本操作。

而PE和LE相当于卷组和逻辑卷的块，卷组大小=PE的大小xPE的数量，同理，逻辑卷的大小=LE的大小xLE的数量，PE默认4M，LE大小不知道是继承PE还是不继承，反正PE默认4M的情况下，LE默认也是4M。小了影响性能，大了占空间，没有专一用途，还是默认不设最好。

## 逻辑卷的创建与使用

首先创建要加入逻辑卷的分区，这里是新添加的sdb盘，整盘分一个分区

```shell
linux-wugz:/home/lengqing # fdisk /dev/sdb

Welcome to fdisk (util-linux 2.33.1).
Changes will remain in memory only, until you decide to write them.
Be careful before using the write command.

Device does not contain a recognized partition table.
Created a new DOS disklabel with disk identifier 0x2f5191a8.

Command (m for help): g
Created a new GPT disklabel (GUID: 8B7DFDAA-0514-0146-9F52-1FDCF9F3D6E5).

Command (m for help): n
Partition number (1-128, default 1): 1
First sector (2048-41943006, default 2048):
Last sector, +/-sectors or +/-size{K,M,G,T,P} (2048-41943006, default 41943006):

Created a new partition 1 of type 'Linux filesystem' and of size 20 GiB.

Command (m for help): t
Selected partition 1
Partition type (type L to list all types): L

Partition type (type L to list all types): 31
Changed type of partition 'Linux filesystem' to 'Linux LVM'.

Command (m for help): p
Disk /dev/sdb: 20 GiB, 21474836480 bytes, 41943040 sectors
Disk model: VMware Virtual S
Units: sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes
Disklabel type: gpt
Disk identifier: 8B7DFDAA-0514-0146-9F52-1FDCF9F3D6E5

Device     Start      End  Sectors Size Type
/dev/sdb1   2048 41943006 41940959  20G Linux LVM

Command (m for help): w                
The partition table has been altered.
Calling ioctl() to re-read partition table.
Syncing disks.
```

然后将sdb1分区转化为物理卷

```shell
linux-wugz:/home/lengqing # pvcreate /dev/sdb1
Physical volume "/dev/sdb1" successfully created.
```

使用物理卷/dev/sdb1创建名为VG1的卷组

```shell
linux-wugz:/home/lengqing # vgcreate VG1 /dev/sdb1
Volume group "VG1" successfully created
```

然后可以查看已有的卷组，我这里装系统就用了LVM，所以有个系统默认的system卷组

```shell
linux-wugz:/home/lengqing # vgs
  VG     #PV #LV #SN Attr   VSize  VFree
  VG1      1   0   0 wz--n- 20.00g 20.00g
  system   1   1   0 wz--n- 15.99g     0
linux-wugz:/home/lengqing # vgdisplay
  --- Volume group ---
  VG Name               system
  System ID
  Format                lvm2
  Metadata Areas        1
  Metadata Sequence No  2
  VG Access             read/write
  VG Status             resizable
  MAX LV                0
  Cur LV                1
  Open LV               1
  Max PV                0
  Cur PV                1
  Act PV                1
  VG Size               15.99 GiB
  PE Size               4.00 MiB
  Total PE              4093
  Alloc PE / Size       4093 / 15.99 GiB
  Free  PE / Size       0 / 0
  VG UUID               BfPwXl-TBTe-N6Rd-NaJT-xU2T-Ce0V-HqIN8L

  --- Volume group ---
  VG Name               VG1
  System ID
  Format                lvm2
  Metadata Areas        1
  Metadata Sequence No  1
  VG Access             read/write
  VG Status             resizable
  MAX LV                0
  Cur LV                0
  Open LV               0
  Max PV                0
  Cur PV                1
  Act PV                1
  VG Size               20.00 GiB
  PE Size               4.00 MiB
  Total PE              5119
  Alloc PE / Size       0 / 0
  Free  PE / Size       5119 / 20.00 GiB
  VG UUID               ojpEne-jsFm-uB6R-azHU-zJmt-wP46-JvFeNc
```

这里是4M的PE，总共5119个PE，离显示的20.00GiB还差4MiB

然后在VG1上创建名为LV1的逻辑卷

```shell
linux-wugz:/home/lengqing # lvcreate -l 5119 -n LV1 VG1
  Logical volume "LV1" created.
```
这里使用-l的参数，直接分配PE数量，也可以使用-L的参数，直接分配空间大小。但是实际可分配的空间不足以分配，比如显示20.00GiB，实际可分配空间只有5119x4MiB=20476MiB=19.996GiB，直接分配20G就会报错。

到这里就已经创建好逻辑卷了，查看一下，这里还是有一个是系统创建的逻辑卷，叫root

```shell
linux-wugz:/home/lengqing # lvs
  LV   VG     Attr       LSize  Pool Origin Data%  Meta%  Move Log Cpy%Sync Convert
  LV1  VG1    -wi-a----- 20.00g
  root system -wi-ao---- 15.99g
linux-wugz:/home/lengqing # lvdisplay
  --- Logical volume ---
  LV Path                /dev/system/root
  LV Name                root
  VG Name                system
  LV UUID                fsZvJJ-NCk9-rHvR-rSOh-80tK-9GxZ-H5BRSw
  LV Write Access        read/write
  LV Creation host, time install, 2019-10-04 00:06:45 +0800
  LV Status              available
  # open                 1
  LV Size                15.99 GiB
  Current LE             4093
  Segments               1
  Allocation             inherit
  Read ahead sectors     auto
  - currently set to     1024
  Block device           254:0

  --- Logical volume ---
  LV Path                /dev/VG1/LV1
  LV Name                LV1
  VG Name                VG1
  LV UUID                xAccOh-PRJd-jtuR-jx7F-F9Qr-N3nw-Cxm7yh
  LV Write Access        read/write
  LV Creation host, time linux-wugz, 2019-12-02 02:22:10 +0800
  LV Status              available
  # open                 0
  LV Size                20.00 GiB
  Current LE             5119
  Segments               1
  Allocation             inherit
  Read ahead sectors     auto
  - currently set to     1024
  Block device           254:1

```

然后对逻辑卷LV1进行格式化与挂载，需要注意的是xfs只能扩容，不能缩容，建议使用ext4

```shell
linux-wugz:/home/lengqing # mkfs.xfs /dev/VG1/LV1
meta-data=/dev/VG1/LV1           isize=512    agcount=4, agsize=1310464 blks
         =                       sectsz=512   attr=2, projid32bit=1
         =                       crc=1        finobt=1, sparse=0, rmapbt=0
         =                       reflink=0
data     =                       bsize=4096   blocks=5241856, imaxpct=25
         =                       sunit=0      swidth=0 blks
naming   =version 2              bsize=4096   ascii-ci=0, ftype=1
log      =internal log           bsize=4096   blocks=2560, version=2
         =                       sectsz=512   sunit=0 blks, lazy-count=1
realtime =none                   extsz=4096   blocks=0, rtextents=0
linux-wugz:/home/lengqing # mkdir /mnt/lvm
linux-wugz:/home/lengqing # mount /dev/VG1/LV1 /mnt/lvm
linux-wugz:/home/lengqing # df -h
Filesystem               Size  Used Avail Use% Mounted on
devtmpfs                 2.0G  8.0K  2.0G   1% /dev
tmpfs                    2.0G     0  2.0G   0% /dev/shm
tmpfs                    2.0G  9.4M  2.0G   1% /run
tmpfs                    2.0G     0  2.0G   0% /sys/fs/cgroup
/dev/mapper/system-root   16G  7.5G  8.6G  47% /
tmpfs                    394M   16K  394M   1% /run/user/1000
/dev/mapper/VG1-LV1       20G   53M   20G   1% /mnt/lvm
```

要想开机挂载，在`/etc/fstab`中加一行就可以了

> /dev/mapper/VG1-LV1  /mnt/lvm  xfs defaults  0  1

到这里为止，逻辑卷就可以正常用了，往其中添加一份文件，检查卸载逻辑卷后文件是不是不在了，以及后期逻辑卷的调整后此文件是不是还在

```shell
linux-wugz:/home/lengqing # echo 'lengqinglvm' > /mnt/lvm/lengqinglvm
linux-wugz:/home/lengqing # cat /mnt/lvm/lengqinglvm
lengqinglvm
```



## 逻辑卷的扩容

对新加入的磁盘sdc创建分区sdc1，转化物理卷，并添加到卷组VG1

```shell
linux-wugz:/home/lengqing # fdisk /dev/sdc

Welcome to fdisk (util-linux 2.33.1).
Changes will remain in memory only, until you decide to write them.
Be careful before using the write command.

Device does not contain a recognized partition table.
Created a new DOS disklabel with disk identifier 0xe5b9393b.

Command (m for help): g
Created a new GPT disklabel (GUID: F8A76081-C61D-6A41-A62E-F39B89C0D886).

Command (m for help): n
Partition number (1-128, default 1):
First sector (2048-41943006, default 2048):
Last sector, +/-sectors or +/-size{K,M,G,T,P} (2048-41943006, default 41943006):

Created a new partition 1 of type 'Linux filesystem' and of size 20 GiB.

Command (m for help): t
Selected partition 1
Partition type (type L to list all types): 31
Changed type of partition 'Linux filesystem' to 'Linux LVM'.

Command (m for help): w
The partition table has been altered.
Calling ioctl() to re-read partition table.
Syncing disks.

linux-wugz:/home/lengqing # fdisk /dev/sdc -l
Disk /dev/sdc: 20 GiB, 21474836480 bytes, 41943040 sectors
Disk model: VMware Virtual S
Units: sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes
Disklabel type: gpt
Disk identifier: F8A76081-C61D-6A41-A62E-F39B89C0D886

Device     Start      End  Sectors Size Type
/dev/sdc1   2048 41943006 41940959  20G Linux LVM
linux-wugz:/home/lengqing # pvcreate /dev/sdc1
  Physical volume "/dev/sdc1" successfully created.
linux-wugz:/home/lengqing # vgextend VG1 /dev/sdc1
  Volume group "VG1" successfully extended
```

此时查看卷组VG1，此时大小已经变为了39.99G，剩余空间20.00G

```shell
linux-wugz:/home/lengqing # vgs
  VG     #PV #LV #SN Attr   VSize  VFree
  VG1      2   1   0 wz--n- 39.99g 20.00g
  system   1   1   0 wz--n- 15.99g     0
linux-wugz:/home/lengqing # vgdisplay
  --- Volume group ---
  VG Name               system
  System ID
  Format                lvm2
  Metadata Areas        1
  Metadata Sequence No  2
  VG Access             read/write
  VG Status             resizable
  MAX LV                0
  Cur LV                1
  Open LV               1
  Max PV                0
  Cur PV                1
  Act PV                1
  VG Size               15.99 GiB
  PE Size               4.00 MiB
  Total PE              4093
  Alloc PE / Size       4093 / 15.99 GiB
  Free  PE / Size       0 / 0
  VG UUID               BfPwXl-TBTe-N6Rd-NaJT-xU2T-Ce0V-HqIN8L

  --- Volume group ---
  VG Name               VG1
  System ID
  Format                lvm2
  Metadata Areas        2
  Metadata Sequence No  3
  VG Access             read/write
  VG Status             resizable
  MAX LV                0
  Cur LV                1
  Open LV               1
  Max PV                0
  Cur PV                2
  Act PV                2
  VG Size               39.99 GiB
  PE Size               4.00 MiB
  Total PE              10238
  Alloc PE / Size       5119 / 20.00 GiB
  Free  PE / Size       5119 / 20.00 GiB
  VG UUID               ojpEne-jsFm-uB6R-azHU-zJmt-wP46-JvFeNc

```

将剩余空间分配到逻辑卷LV1，注意`+`不能省，加号代表加空间，而不带加号代表就分配这么多

```shell
linux-wugz:/home/lengqing # lvextend -l +5119 /dev/VG1/LV1
  Size of logical volume VG1/LV1 changed from 20.00 GiB (5119 extents) to 39.99 GiB (10238 extents).
  Logical volume VG1/LV1 successfully resized.
```

此时逻辑卷文件系统仍然是20G，需要调整文件系统大小才会改变

```shell
linux-wugz:/home/lengqing # xfs_growfs /mnt/lvm/
meta-data=/dev/mapper/VG1-LV1    isize=512    agcount=4, agsize=1310464 blks
         =                       sectsz=512   attr=2, projid32bit=1
         =                       crc=1        finobt=1 spinodes=0 rmapbt=0
         =                       reflink=0
data     =                       bsize=4096   blocks=5241856, imaxpct=25
         =                       sunit=0      swidth=0 blks
naming   =version 2              bsize=4096   ascii-ci=0 ftype=1
log      =internal               bsize=4096   blocks=2560, version=2
         =                       sectsz=512   sunit=0 blks, lazy-count=1
realtime =none                   extsz=4096   blocks=0, rtextents=0
data blocks changed from 5241856 to 10483712
linux-wugz:/home/lengqing # df -h
Filesystem               Size  Used Avail Use% Mounted on
devtmpfs                 2.0G  8.0K  2.0G   1% /dev
tmpfs                    2.0G     0  2.0G   0% /dev/shm
tmpfs                    2.0G  9.4M  2.0G   1% /run
tmpfs                    2.0G     0  2.0G   0% /sys/fs/cgroup
/dev/mapper/system-root   16G  7.1G  8.9G  45% /
tmpfs                    394M   12K  394M   1% /run/user/1000
/dev/mapper/VG1-LV1       40G   74M   40G   1% /mnt/lvm
```

需要注意的是xfs扩容只能在挂载的情况下，命令后面只能跟挂载点，不能直接跟设备名，例如这样

```shell
linux-wugz:/home/lengqing # xfs_growfs /dev/mapper/VG1-LV1
xfs_growfs: /dev/mapper/VG1-LV1 is not a mounted XFS filesystem
```

之前创建的文件仍然正常

```shell
linux-wugz:/home/lengqing # cat /mnt/lvm/lengqinglvm
lengqinglvm
```



## 逻辑卷的缩容

由于xfs不能缩容，所以还得换ext4来演示（(⊙﹏⊙)早知道就不用xfs了）

```shell
linux-wugz:/home/lengqing # umount /mnt/lvm
linux-wugz:/home/lengqing # mkfs.ext4 /dev/mapper/VG1-LV1
mke2fs 1.43.8 (1-Jan-2018)
/dev/mapper/VG1-LV1 contains a xfs file system
Proceed anyway? (y,N) y
Creating filesystem with 10483712 4k blocks and 2621440 inodes
Filesystem UUID: cb51dc65-ffb4-4180-b886-ddec1a723a3b
Superblock backups stored on blocks:
        32768, 98304, 163840, 229376, 294912, 819200, 884736, 1605632, 2654208,
        4096000, 7962624

Allocating group tables: done
Writing inode tables: done
Creating journal (65536 blocks): done
Writing superblocks and filesystem accounting information: done

linux-wugz:/home/lengqing # mount /dev/mapper/VG1-LV1 /mnt/lvm/
```

要开机挂载的还得改一下`/etc/fstab`

> /dev/mapper/VG1-LV1  /mnt/lvm  ext4 defaults  0  1

然后还是添加一份文件，检查缩容后文件是否正常

```shell
linux-wugz:/home/lengqing # echo 'lengqinglvmext4' > /mnt/lvm/lengqinglvmext4
linux-wugz:/home/lengqing # cat /mnt/lvm/lengqinglvmext4
lengqinglvmext4
```

卸载分区并检查文件系统，卸载时工作目录不能在挂载点内

```shell
linux-wugz:/home/lengqing #  umount /mnt/lvm
linux-wugz:/home/lengqing # e2fsck -f /dev/mapper/VG1-LV1
e2fsck 1.43.8 (1-Jan-2018)
Pass 1: Checking inodes, blocks, and sizes
Pass 2: Checking directory structure
Pass 3: Checking directory connectivity
Pass 4: Checking reference counts
Pass 5: Checking group summary information
/dev/mapper/VG1-LV1: 12/2621440 files (0.0% non-contiguous), 242383/10483712 blocks
```

测试缩减逻辑卷，`-t`代表测试，需要注意的是只能按PE的整数倍来缩减，因为PE是最小存储单元，不能分配半个或者零点几个，这里PE是4M，按4M的整数倍都可以，注意不带减号是缩减到15G，而带了`-`是缩减15G

```shell
linux-wugz:/home/lengqing # lvreduce -L 15G /dev/VG1/LV1 -t
  TEST MODE: Metadata will NOT be updated and volumes will not be (de)activated.
  WARNING: Reducing active logical volume to 15.00 GiB.
  THIS MAY DESTROY YOUR DATA (filesystem etc.)
Do you really want to reduce VG1/LV1? [y/n]: y
  Size of logical volume VG1/LV1 changed from 39.99 GiB (10238 extents) to 15.00 GiB (3840 extents).
  Logical volume VG1/LV1 successfully resized.
```

测试没有毛病，这里逻辑卷仍然是39.99G

```shell
linux-wugz:/home/lengqing # lvs
  LV   VG     Attr       LSize  Pool Origin Data%  Meta%  Move Log Cpy%Sync Convert
  LV1  VG1    -wi-ao---- 39.99g
  root system -wi-ao---- 15.99g
linux-wugz:/home/lengqing # lvdisplay
  --- Logical volume ---
  LV Path                /dev/system/root
  LV Name                root
  VG Name                system
  LV UUID                fsZvJJ-NCk9-rHvR-rSOh-80tK-9GxZ-H5BRSw
  LV Write Access        read/write
  LV Creation host, time install, 2019-10-04 00:06:45 +0800
  LV Status              available
  # open                 1
  LV Size                15.99 GiB
  Current LE             4093
  Segments               1
  Allocation             inherit
  Read ahead sectors     auto
  - currently set to     1024
  Block device           254:0

  --- Logical volume ---
  LV Path                /dev/VG1/LV1
  LV Name                LV1
  VG Name                VG1
  LV UUID                xAccOh-PRJd-jtuR-jx7F-F9Qr-N3nw-Cxm7yh
  LV Write Access        read/write
  LV Creation host, time linux-wugz, 2019-12-02 02:22:10 +0800
  LV Status              available
  # open                 1
  LV Size                39.99 GiB
  Current LE             10238
  Segments               2
  Allocation             inherit
  Read ahead sectors     auto
  - currently set to     1024
  Block device           254:1

```

首先调整文件系统，先缩减逻辑卷到比文件系统小，文件系统大概也保不住

```shell
linux-wugz:/home/lengqing # resize2fs /dev/mapper/VG1-LV1 15G
resize2fs 1.43.8 (1-Jan-2018)
Resizing the filesystem on /dev/mapper/VG1-LV1 to 3932160 (4k) blocks.
The filesystem on /dev/mapper/VG1-LV1 is now 3932160 (4k) blocks long.

```

挂载分区并查看，文件系统已经是15G，之前的文件也没有问题

```shell
linux-wugz:/home/lengqing # mount -a
linux-wugz:/home/lengqing # df -h
Filesystem               Size  Used Avail Use% Mounted on
devtmpfs                 2.0G  8.0K  2.0G   1% /dev
tmpfs                    2.0G     0  2.0G   0% /dev/shm
tmpfs                    2.0G  9.4M  2.0G   1% /run
tmpfs                    2.0G     0  2.0G   0% /sys/fs/cgroup
/dev/mapper/system-root   16G  7.1G  8.9G  45% /
tmpfs                    394M   12K  394M   1% /run/user/1000
/dev/mapper/VG1-LV1       15G   41M   14G   1% /mnt/lvm
linux-wugz:/home/lengqing # cat /mnt/lvm/l
lengqinglvmext4  lost+found/
linux-wugz:/home/lengqing # cat /mnt/lvm/l
lengqinglvmext4  lost+found/
linux-wugz:/home/lengqing # cat /mnt/lvm/lengqinglvmext4
lengqinglvmext4
```

重新卸载分区并检查文件系统，缩减逻辑卷至15G，这次没有`-t`的参数

```shell
linux-wugz:/home/lengqing # umount /mnt/lvm
linux-wugz:/home/lengqing # e2fsck -f /dev/mapper/VG1-LV1
e2fsck 1.43.8 (1-Jan-2018)
Pass 1: Checking inodes, blocks, and sizes
Pass 2: Checking directory structure
Pass 3: Checking directory connectivity
Pass 4: Checking reference counts
Pass 5: Checking group summary information
/dev/mapper/VG1-LV1: 12/983040 files (0.0% non-contiguous), 137494/3932160 blocks
linux-wugz:/home/lengqing # lvreduce -L 15G /dev/VG1/LV1
  WARNING: Reducing active logical volume to 15.00 GiB.
  THIS MAY DESTROY YOUR DATA (filesystem etc.)
Do you really want to reduce VG1/LV1? [y/n]: y
  Size of logical volume VG1/LV1 changed from 39.99 GiB (10238 extents) to 15.00 GiB (3840 extents).
  Logical volume VG1/LV1 successfully resized.
```

此时查看逻辑卷，LV1已经变成了15G

```shell
linux-wugz:/home/lengqing # lvs
  LV   VG     Attr       LSize  Pool Origin Data%  Meta%  Move Log Cpy%Sync Convert
  LV1  VG1    -wi-a----- 15.00g
  root system -wi-ao---- 15.99g
linux-wugz:/home/lengqing # lvdisplay
  --- Logical volume ---
  LV Path                /dev/system/root
  LV Name                root
  VG Name                system
  LV UUID                fsZvJJ-NCk9-rHvR-rSOh-80tK-9GxZ-H5BRSw
  LV Write Access        read/write
  LV Creation host, time install, 2019-10-04 00:06:45 +0800
  LV Status              available
  # open                 1
  LV Size                15.99 GiB
  Current LE             4093
  Segments               1
  Allocation             inherit
  Read ahead sectors     auto
  - currently set to     1024
  Block device           254:0

  --- Logical volume ---
  LV Path                /dev/VG1/LV1
  LV Name                LV1
  VG Name                VG1
  LV UUID                xAccOh-PRJd-jtuR-jx7F-F9Qr-N3nw-Cxm7yh
  LV Write Access        read/write
  LV Creation host, time linux-wugz, 2019-12-02 02:22:10 +0800
  LV Status              available
  # open                 0
  LV Size                15.00 GiB
  Current LE             3840
  Segments               1
  Allocation             inherit
  Read ahead sectors     auto
  - currently set to     1024
  Block device           254:1
```

查看卷组VG1，已分配空间变成了15G，剩余空间还有24.99G

```shell
linux-wugz:/home/lengqing # vgdisplay
  --- Volume group ---
  VG Name               system
  System ID
  Format                lvm2
  Metadata Areas        1
  Metadata Sequence No  2
  VG Access             read/write
  VG Status             resizable
  MAX LV                0
  Cur LV                1
  Open LV               1
  Max PV                0
  Cur PV                1
  Act PV                1
  VG Size               15.99 GiB
  PE Size               4.00 MiB
  Total PE              4093
  Alloc PE / Size       4093 / 15.99 GiB
  Free  PE / Size       0 / 0
  VG UUID               BfPwXl-TBTe-N6Rd-NaJT-xU2T-Ce0V-HqIN8L

  --- Volume group ---
  VG Name               VG1
  System ID
  Format                lvm2
  Metadata Areas        2
  Metadata Sequence No  5
  VG Access             read/write
  VG Status             resizable
  MAX LV                0
  Cur LV                1
  Open LV               0
  Max PV                0
  Cur PV                2
  Act PV                2
  VG Size               39.99 GiB
  PE Size               4.00 MiB
  Total PE              10238
  Alloc PE / Size       3840 / 15.00 GiB
  Free  PE / Size       6398 / 24.99 GiB
  VG UUID               ojpEne-jsFm-uB6R-azHU-zJmt-wP46-JvFeNc

```

挂载查看，文件系统正常，文件也还在

```shell
linux-wugz:/home/lengqing # mount -a
linux-wugz:/home/lengqing # df -h
Filesystem               Size  Used Avail Use% Mounted on
devtmpfs                 2.0G  8.0K  2.0G   1% /dev
tmpfs                    2.0G     0  2.0G   0% /dev/shm
tmpfs                    2.0G  9.4M  2.0G   1% /run
tmpfs                    2.0G     0  2.0G   0% /sys/fs/cgroup
/dev/mapper/system-root   16G  7.1G  8.9G  45% /
tmpfs                    394M   12K  394M   1% /run/user/1000
/dev/mapper/VG1-LV1       15G   41M   14G   1% /mnt/lvm
linux-wugz:/home/lengqing # cat /mnt/lvm/lengqinglvmext4
lengqinglvmext4
```

## 移除物理磁盘

移除之前，需要移除的磁盘不能有数据，所以需要将数据转移到其它磁盘上，其它磁盘需要足够大能容纳已分配的空间，然后转移完成就可以移除了。

这里移除先添加的磁盘sdb，因为数据肯定在它上面。~~（我猜的，管它在哪个上面，我就是要移除这块）~~

将物理卷sdb1上的数据转移到物理卷sdc1上

```shell
linux-wugz:/home/lengqing # pvmove /dev/sdb1 /dev/sdc1
  /dev/sdb1: Moved: 4.90%
  /dev/sdb1: Moved: 100.00% 
```

完成之后，从卷组VG1中移除物理卷/dev/sdb1

```shell
linux-wugz:/home/lengqing # vgreduce VG1 /dev/sdb1
  Removed "/dev/sdb1" from volume group "VG1"
```

检查挂载以及文件，看起来完全正常

```shell
linux-wugz:/home/lengqing # df -hT
Filesystem              Type      Size  Used Avail Use% Mounted on
devtmpfs                devtmpfs  2.0G  8.0K  2.0G   1% /dev
tmpfs                   tmpfs     2.0G     0  2.0G   0% /dev/shm
tmpfs                   tmpfs     2.0G  9.4M  2.0G   1% /run
tmpfs                   tmpfs     2.0G     0  2.0G   0% /sys/fs/cgroup
/dev/mapper/system-root xfs        16G  7.1G  8.9G  45% /
tmpfs                   tmpfs     394M   12K  394M   1% /run/user/1000
/dev/mapper/VG1-LV1     ext4       15G   41M   14G   1% /mnt/lvm
linux-wugz:/home/lengqing # cat /mnt/lvm/lengqinglvmext4
lengqinglvmext4
```
其实可以先查看物理卷的占用（我这里已经移除了sdb1，但也能看出来sdc1现在总共20G，用了15G，5G剩余）

```shell
linux-wugz:/home/lengqing # pvscan
  PV /dev/sda2   VG system          lvm2 [15.99 GiB / 0    free]
  PV /dev/sdc1   VG VG1             lvm2 [20.00 GiB / 5.00 GiB free]
  PV /dev/sdb1                      lvm2 [20.00 GiB]
  Total: 3 [55.98 GiB] / in use: 2 [35.98 GiB] / in no VG: 1 [20.00 GiB]
```

## 替换物理磁盘

替换磁盘=先加入磁盘，再移除磁盘。

方法和步骤前面都有。

# 完