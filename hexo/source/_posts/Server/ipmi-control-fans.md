---
layout: post
title:  "使用ipmitool控制戴尔R510服务器风扇转速"
date:   2019-10-19 17:19:24 +0800
categories: Server
---
> 注意留意服务器组件的温度，手动控制可能导致高温过热，但是11G服务器的ipmi功能太过简单，无法获取温度，无法通过脚本获取温度自动调节转速。
> 重启服务器需要重新设置。

### 0. 安装ipmitool
Debian下

```shell
apt-get install ipmitool
```

### 1. 关闭自动风扇控制

```shell
ipmitool -I lanplus -U idrac用户名 -P idrac密码 -H idarc地址 raw 0x30 0x30 0x01 0x00
```

### 2. 调节风扇转速，其中最后一位为转速百分比的16进制，例如`0x14`代表20%，`0x32`为50%，`0x64`为100%
```shell
ipmitool -I lanplus -U idrac用户名 -P idrac密码 -H idarc地址 raw 0x30 0x30 0x02 0xff 0x14
```

### 3. 如果不想手动控制可以重新打开自动风扇控制

   ```shell
ipmitool -I lanplus -U idrac用户名 -P idrac密码 -H idarc地址 raw 0x30 0x30 0x01 0x01
   ```

### 完