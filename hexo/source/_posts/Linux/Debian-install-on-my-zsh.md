---
layout: post
title:  "Debian 安装 zsh 及 oh-my-zsh"
date:   2018-07-29 20:02:00 +0800
categories: Linux
---
### 安装 zsh 以及 git （ oh-my-zsh 需要）
```shell
apt-get install zsh git 
```
### 切换默认 SH
```shell
chsh -s /bin/zsh
```
### 安装 oh-my-zsh (反正我这里 raw.github.com 连线质量比较差，下了好几次才好)
```shell
sh -c "$(wget https://raw.githubusercontent.com/robbyrussell/oh-my-zsh/master/tools/install.sh -O -)"
```
### 更改主题
```shell
nano ~/.zshrc
```
其中有一行 
> ZSH_THEME="robbyrussell"

将其中的  robbyrussell 改成你想要的主题，或者 random ， random 是随机主题，每次登录随机换一个主题。

