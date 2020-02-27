---
layout: post
title:  "Github使用GPG密钥commit"
date:   2019-08-28 22:25:00 +0800
updated: 2019-10-27 11:26:00 +0800
categories: Linux
---
参考文档：[[git]使用GPG签名你的commit](https://www.cnblogs.com/xueweihan/p/5430451.html)

在[上一篇文章](https://blog.lengqing.org/2019/08/28/Linux/GPG/ )中创建了GPG密钥之后

## 0 Github端

首先在 [https://github.com/settings/keys](https://github.com/settings/keys) 点击绿色的`New GPG key`

将GPG密钥的公钥内容输入进去，然后点击绿色的`Add GPG Key`



## 1 本地端

添加GPG密钥

```shell
git config --global user.signingkey FD88ED069146EA37
```

全局启用

```shell
git config --global commit.gpgsign true #开启
git config --global commit.gpgsign false #关闭
```

仅在单个仓库中开启，进入仓库目录，然后运行

```shell
git config commit.gpgsign true #开启
git config commit.gpgsign false #关闭
```


然后commit之后push之后即可在远程仓库的commits中看到绿色的"Verified"标记

