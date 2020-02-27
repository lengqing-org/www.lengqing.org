---
layout: post
title:  "在Dell iDRAC里导入自己的SSL证书"
date:   2019-04-21 01:07:00 +0800
categories: Server
---
主要内容转载自：[在Dell iDRAC里导入wildcard SSL证书的方法](https://blog.vsean.net/post/186)以及 [为 Dell iDRAC8 上传 SSL 证书](https://blog.nanpuyue.com/2018/039.html)。
琢磨了好久要换证书了，主要是因为自带的10年戴尔证书不受Java信任，Java8开始的版本因为安全政策问题没法用虚拟控制台，一直用的Java7，不敢更新，一更新就不让连。。。

**（然而换好证书也并不能，还得用jre7，推测这个证书只是web页面的证书，虚拟控制台还是用的戴尔的数字签名）**

**要先在idrac的“网页界面-iDRAC 设置-服务”中启用“远程 RACADM”，否则就没法弄，网上的教程都不讲这个的，弄得我总失败。**

### 0 从戴尔官网下载racadm工具包，Linux和Windows平台的都有

然后安装

### 1 将iDRAC的证书长度扩展到2048bit(但我没弄这个好像也没啥问题)

```shell
racadm -r [server ip] -i config -g cfgRacSecurity -o cfgRacSecCsrKeySize 2048
```

### 2 上传私钥

```shell
racadm -r [server ip] -i sslkeyupload -t 1 -f 私钥.key
```

### 3 上传证书（iDRAC只接受X509 Base64编码的证书，不接受二进制编码(DER)证书）

```shell
racadm -r [server ip] -i sslcertupload -t 1 -f 证书.crt
```

### 4 使证书生效（我的自己重启了，没用上这一步）

```shell
racadm -r [server ip] -i racreset
```

