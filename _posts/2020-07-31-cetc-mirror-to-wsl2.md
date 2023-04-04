---
layout: post
title: cetc-mirror-to-wsl2
postTitle: 智能博弈挑战赛镜像迁移到WSL 2中运行
categories: [WSL, CETC, docker, Environment]
description: 
keywords: WSL, CETC, docker, Environment
mathjax: false
typora-root-url: ..
---

[比赛地址](https://www.dcjingsai.com/v2/cmptDetail.html?id=377)

这比赛的镜像和数据体量都很大，而且后端在Linux跑，前端在Windows上跑。官方文档是要求开VMware虚拟机，这就不太能顶得住。于是我摸索了一个把后端放到WSL 2上跑的操作步骤，可以节约不少系统资源。

## 安装WSL 2和Windows Terminal

注意WSL 2仅支持Win10 2004以上的版本，需要先把Win10更新到2004：

[安装WSL 2微软官方文档](https://docs.microsoft.com/zh-cn/windows/wsl/install-win10#update-to-wsl-2)

Ubuntu版本选择18.04 LTS。

Microsoft Store中安装Windows Terminal，用它操作Ubuntu。

![](https://i.loli.net/2020/08/01/xITVc9y7wmvPlCU.png)

## WSL迁移到非系统盘

镜像包非常大，WSL放在系统盘会对机器造成很大压力，因此要把WSL迁移到其他盘：

[WSL 2迁移步骤](https://www.jskap.com/notes/how-to-move-wsl2-disto/)

注意，无论下载的`LxRunOffline`是新版还是旧版，都一定要按照旧版的方式操作：先转成WSL 1，然后再迁移，最后转回WSL 2。直接以WSL 2迁移可能会失败。

*可选*：在VS Code中安装插件`Remote - WSL`，然后点击左下角的`><`图标，随便在WSL中打开一个文件夹，等待连接完成。连接完成后，就能在WSL中用`code`命令打开文件了。不过`code`不能`sudo`，需要管理员权限的地方还是要`vim`。

## apt-get换源

```shell
cd /etc/apt && sudo cp sources.list sources.list.backup && sudo vim sources.list 
```

全删除后输入（代码来自[清华源](https://mirrors.tuna.tsinghua.edu.cn/help/ubuntu/)，注意把官方代码第一段的注释都去掉）：

```shell
deb https://mirrors.tuna.tsinghua.edu.cn/ubuntu/ bionic main restricted universe multiverse
deb-src https://mirrors.tuna.tsinghua.edu.cn/ubuntu/ bionic main restricted universe multiverse
deb https://mirrors.tuna.tsinghua.edu.cn/ubuntu/ bionic-updates main restricted universe multiverse
deb-src https://mirrors.tuna.tsinghua.edu.cn/ubuntu/ bionic-updates main restricted universe multiverse
deb https://mirrors.tuna.tsinghua.edu.cn/ubuntu/ bionic-backports main restricted universe multiverse
deb-src https://mirrors.tuna.tsinghua.edu.cn/ubuntu/ bionic-backports main restricted universe multiverse
deb https://mirrors.tuna.tsinghua.edu.cn/ubuntu/ bionic-security main restricted universe multiverse
deb-src https://mirrors.tuna.tsinghua.edu.cn/ubuntu/ bionic-security main restricted universe multiverse
```

（也可以用[阿里源](https://developer.aliyun.com/mirror/ubuntu)）

保存后更新并安装pip3和rpyc：

```shell
sudo apt-get update
sudo apt-get install python3-pip
pip3 install rpyc
```

如果pip3无法安装，还需要换掉pip3的源（永久修改）：

[更换pip3国内镜像](https://blog.csdn.net/yuzaipiaofei/article/details/80891108)

## 安装docker

WSL 2支持跑docker，安装步骤如下：

[WSL 2安装docker](https://www.cnblogs.com/yunfeifei/p/13158845.html)

如果最后docker运行失败，需要先

```shell
sudo adduser $USER docker
```

然后重启Windows计算机。

## 运行docker镜像

把下载的镜像`combatmodserver_v1.2.tar`导入到docker中：

```shell
docker load < combatmodserver_v1.2.tar
```

即可运行：

```shell
python3 run.py
```

最后获取连接到仿真平台的IP地址：

```shell
ifconfig -a
```

![](https://i.loli.net/2020/08/01/OnCYSm8Pa7fHFb2.png)