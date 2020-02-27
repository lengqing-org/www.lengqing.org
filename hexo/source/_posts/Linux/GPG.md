---
layout: post
title:  "GPG密钥的创建与简单管理"
date:   2019-08-28 20:53:00 +0800
categories: Linux
---
参考文档：[GPG入门](https://www.jianshu.com/p/1257dbf3ed8e)、[GPG密钥的生成与使用](https://www.jianshu.com/p/7f19ceacf57c)

## 0 安装GPG软件
一般Linux发行版都自带了的，就不讲了，

我目前使用的OpenSuSE Leap 15.1的KDE Plasma环境自带了Kleopatra，Kleopatra是一个能用的图形化GPG密钥管理器，可用于导出绝密密钥。

命令行GPG的用法：

```shell
gpg --help
gpg (GnuPG) 2.2.5
libgcrypt 1.8.2
Copyright (C) 2018 Free Software Foundation, Inc.
License GPLv3+: GNU GPL version 3 or later <https://gnu.org/licenses/gpl.html>
This is free software: you are free to change and redistribute it.
There is NO WARRANTY, to the extent permitted by law.

Home: /home/lengqing/.gnupg
支持的算法：
公钥：RSA, ELG, DSA, ECDH, ECDSA, EDDSA
对称加密：IDEA, 3DES, CAST5, BLOWFISH, AES, AES192, AES256,
     TWOFISH, CAMELLIA128, CAMELLIA192, CAMELLIA256
散列：SHA1, RIPEMD160, SHA256, SHA384, SHA512, SHA224
压缩：不压缩, ZIP, ZLIB, BZIP2

Syntax: gpg [options] [files]
Sign, check, encrypt or decrypt
Default operation depends on the input data

指令：
 
 -s, --sign                  make a signature
     --clear-sign            make a clear text signature
 -b, --detach-sign           生成一份分离的签名
 -e, --encrypt               加密数据
 -c, --symmetric             仅使用对称加密
 -d, --decrypt               解密数据(默认)
     --verify                验证签名
 -k, --list-keys             列出密钥
     --list-signatures       列出密钥和签名
     --check-signatures      列出并检查密钥签名
     --fingerprint           列出密钥和指纹
 -K, --list-secret-keys      列出私钥
     --generate-key          生成一副新的密钥对
     --quick-generate-key    quickly generate a new key pair
     --quick-add-uid         quickly add a new user-id
     --quick-revoke-uid      quickly revoke a user-id
     --quick-set-expire      quickly set a new expiration date
     --full-generate-key     full featured key pair generation
     --generate-revocation   生成一份吊销证书
     --delete-keys           从公钥钥匙环里删除密钥
     --delete-secret-keys    从私钥钥匙环里删除密钥
     --quick-sign-key        quickly sign a key
     --quick-lsign-key       quickly sign a key locally
     --sign-key              为某把密钥添加签名
     --lsign-key             为某把密钥添加本地签名
     --edit-key              编辑某把密钥或为其添加签名
     --change-passphrase     change a passphrase
     --export                导出密钥
     --send-keys             把密钥导出到某个公钥服务器上
     --receive-keys          从公钥服务器上导入密钥
     --search-keys           在公钥服务器上搜寻密钥
     --refresh-keys          从公钥服务器更新所有的本地密钥
     --import                导入/合并密钥
     --card-status           打印卡状态
     --edit-card             更改卡上的数据
     --change-pin            更改卡的 PIN
     --update-trustdb        更新信任度数据库
     --print-md              print message digests
     --server                run in server mode
     --tofu-policy VALUE     set the TOFU policy for a key

选项：
 
 -a, --armor                 输出经 ASCII 封装
 -r, --recipient USER-ID     encrypt for USER-ID
 -u, --local-user USER-ID    use USER-ID to sign or decrypt
 -z N                        set compress level to N (0 disables)
     --textmode              使用标准的文本模式
 -o, --output FILE           write output to FILE
 -v, --verbose               详细模式
 -n, --dry-run               不做任何改变
 -i, --interactive           覆盖前先询问
     --openpgp               行为严格遵循 OpenPGP 定义

(请参考在线说明以获得所有命令和选项的完整清单)

Examples:

 -se -r Bob [file]          sign and encrypt for user Bob
 --clear-sign [file]        make a clear text signature
 --detach-sign [file]       make a detached signature
 --list-keys [names]        show keys
 --fingerprint [names]      show fingerprints

请向 <https://bugs.gnupg.org> 报告程序缺陷。
请向 <zuxyhere@eastday.com> 反映简体中文翻译的问题。


```



## 1 生成GPG密钥

```shell
gpg --full-generate-key
gpg (GnuPG) 2.2.5; Copyright (C) 2018 Free Software Foundation, Inc.
This is free software: you are free to change and redistribute it.
There is NO WARRANTY, to the extent permitted by law.

请选择您要使用的密钥种类：
   (1) RSA and RSA (default)
   (2) DSA and Elgamal
   (3) DSA (仅用于签名)
   (4) RSA (仅用于签名)
您的选择？ 
RSA 密钥长度应在 1024 位与 4096 位之间。
您想要用多大的密钥尺寸？(2048)
您所要求的密钥尺寸是 2048 位
请设定这把密钥的有效期限。
         0 = 密钥永不过期
      <n>  = 密钥在 n 天后过期
      <n>w = 密钥在 n 周后过期
      <n>m = 密钥在 n 月后过期
      <n>y = 密钥在 n 年后过期
密钥的有效期限是？(0) 
密钥永远不会过期
以上正确吗？(y/n)y

You need a user ID to identify your key; the software constructs the user ID
from the Real Name, Comment and Email Address in this form:
    "Heinrich Heine (Der Dichter) <heinrichh@duesseldorf.de>"

真实姓名：lengqing5977
电子邮件地址：root@lengqing.org
注释：
您选定了这个用户标识：
    “lengqing5977 <root@lengqing.org>”

更改姓名(N)、注释(C)、电子邮件地址(E)或确定(O)/退出(Q)？O
我们需要生成大量的随机字节。这个时候您可以多做些琐事(像是敲打键盘、移动
鼠标、读写硬盘之类的)，这会让随机数字发生器有更好的机会获得足够的熵数。
我们需要生成大量的随机字节。这个时候您可以多做些琐事(像是敲打键盘、移动
鼠标、读写硬盘之类的)，这会让随机数字发生器有更好的机会获得足够的熵数。
gpg: 密钥 FD88ED069146EA37 被标记为绝对信任
gpg: revocation certificate stored as '/home/lengqing/.gnupg/openpgp-revocs.d/EE93CE1B2035C64A60914333FD88ED069146EA37.rev'
公钥和私钥已经生成并经签名。

pub   rsa2048 2019-08-28 [SC]
      EE93CE1B2035C64A60914333FD88ED069146EA37
uid                      lengqing5977 <root@lengqing.org>
sub   rsa2048 2019-08-28 [E]
```

## 2 查看GPG密钥

```shell
gpg --list-keys
/home/lengqing/.gnupg/pubring.kbx
---------------------------------
pub   rsa2048 2019-08-28 [SC]
      EE93CE1B2035C64A60914333FD88ED069146EA37
uid           [ 绝对 ] lengqing5977 <root@lengqing.org>
sub   rsa2048 2019-08-28 [E]

```

```shell
gpg --list-keys --keyid-format long
/home/lengqing/.gnupg/pubring.kbx
---------------------------------
pub   rsa2048/FD88ED069146EA37 2019-08-28 [SC]
      EE93CE1B2035C64A60914333FD88ED069146EA37
uid                 [ 绝对 ] lengqing5977 <root@lengqing.org>
sub   rsa2048/1F62C16D9F3F386F 2019-08-28 [E]

```

其中上面的`FD88ED069146EA37`可以用来代替ID，下面就会用到了。



## 3 导出GPG密钥

导出公钥
```shell
gpg --armor --output public.asc --export FD88ED069146EA37
```
导出私钥

```shell
gpg --armor --output private.asc --export-secret-keys FD88ED069146EA37
```

**需要注意的是，这里导出的私钥公钥在其它系统导入之后依旧不能将密钥信任等级提升至“绝密”等级（也就是自己的密钥/绝对信任），将有使用限制，需要利用其它途径导出“绝密密钥”**

## 4 编辑GPG密钥

```shell
gpg --edit-key FD88ED069146EA37
gpg (GnuPG) 2.2.5; Copyright (C) 2018 Free Software Foundation, Inc.
This is free software: you are free to change and redistribute it.
There is NO WARRANTY, to the extent permitted by law.

私钥可用。

sec  rsa2048/FD88ED069146EA37
     创建于：2019-08-28  有效至：永不过期  可用于：SC  
     信任度：绝对        有效性：绝对
ssb  rsa2048/1F62C16D9F3F386F
     创建于：2019-08-28  有效至：永不过期  可用于：E   
[ 绝对 ] (1). lengqing5977 <root@lengqing.org>

gpg> help
quit        离开这个菜单
save        保存并离开
help        显示这份在线说明
fpr         显示密钥指纹
grip        show the keygrip
list        列出密钥和用户标识
uid         选择用户标识 N
key         选择子钥 N
check       检查签名
sign        为所选用户标识添加签名[* 参见下面的相关命令]
lsign       为所选用户标识添加本地签名
tsign       为所选用户标识添加信任签名
nrsign      为所选用户标识添加不可吊销签名
adduid      增加一个用户标识
addphoto    增加一个照片标识
deluid      删除选定的用户标识
addkey      添加一个子钥
addcardkey  在智能卡上添加一把密钥
keytocard   将一把密钥移动到智能卡上
bkuptocard  将备份密钥转移到卡上
delkey      删除选定的子钥
addrevoker  增加一把吊销密钥
delsig      删除所选用户标识上的签名
expire      变更密钥或所选子钥的使用期限
primary     将所选的用户标识设为首选用户标识
pref        列出首选项(专家模式)
showpref    列出首选项(详细模式)
setpref     设定所选用户标识的首选项
keyserver   设定所选用户标识的首选公钥服务器的 URL
notation    为所选用户标识的设定注记
passwd      更改密码
trust       更改信任度
revsig      吊销所选用户标识上的签名
revuid      吊销选定的用户标识
revkey      吊销密钥或选定的子钥
enable      启用密钥
disable     禁用密钥
showphoto   显示选定的照片标识
clean       压缩不可用的用户标识并删除不可用的签名
minimize    压缩不可用的用户标识并删除所有签名

* The 'sign' command may be prefixed with an 'l' for local signatures (lsign),
  a 't' for trust signatures (tsign), an 'nr' for non-revocable signatures
  (nrsign), or any combination thereof (ltsign, tnrsign, etc.).
```



## 5 将GPG公钥上传到公钥服务器（极不推荐的做法）

上传

```shell
gpg --keyserver hkp://subkeys.pgp.net --send-keys FD88ED069146EA37
```

因为上传是永久不可逆的操作，所以还需要创建撤销证书

```shell
gpg -a -o revocation.cert --gen-revoke FD88ED069146EA37

sec  rsa2048/FD88ED069146EA37 2019-08-28 lengqing5977 <root@lengqing.org>

要为这把密钥建立一份吊销证书吗？(y/N)y
请选择吊销的原因：
  0 = 未指定原因
  1 = 密钥已泄漏
  2 = 密钥被替换
  3 = 密钥不再使用
  Q = 取消
(也许您会想要在这里选择 1)
您的决定是什么？0
请输入描述(可选)；以空白行结束：
> 
吊销原因：未指定原因
(不给定描述)
这样可以吗？ (y/N)y
已建立吊销证书。

请把这个文件转移到一个可隐藏起来的介质(如软盘)上；如果坏人能够取得这
份证书的话，那么他就能让您的密钥无法继续使用。把这份凭证打印出来再藏
到安全的地方也是很好的方法，以免您的保存媒体损毁而无法读取。但是千万
小心：您的机器上的打印系统可能会在打印过程中把这些数据临时在某个其他
人也能够看得到的地方！

```



## 6 删除GPG密钥

删除公钥

```shell
gpg --delete-key FD88ED069146EA37
```

删除私钥

```shell
gpg --delete-secret-keys FD88ED069146EA37
```

