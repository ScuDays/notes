---
title: 构建deb软件安装包
date: 2024-12-26 09:52:21
modify: 2024-12-26 09:52:21
author: days
category: Embedded Linux
published: 2024-12-26
draft: false
description: 构建deb软件安装包
tags: [deb包, Linux软件包, dpkg工具]
---
# 构建deb软件安装包
## Linux 软件包的组成

| 文件类型     | 保存目录       |
| ------------ | -------------- |
| 普通程序     | /usr/bin       |
| root权限程序 | /usr/sbin      |
| 程序配置文件 | /etc           |
| 日志文件     | /var/log       |
| 文档文件     | /usr/share/doc |

软件包分类

- 源码包

  优点:

  - 开源免费

  - 自由裁剪功能
  - 修改源代码

  缺点:

  - 安装步骤繁琐
  - 编译时间长
  - 新手无法解决编译问题

- 二进制包

  优点:

  - 简单易用
  - 安装速度快

  缺点:

  - 无法阅读修改源码
  - 无法裁剪功能
  - 依赖性强

### deb包
#### 概念

Debian、Ubuntu、Deepin等Linux发行版的软件安装包。

#### 结构

Deb 软件包里面的结构：它具有 DEBIAN 和软件具体安装目录（如 etc, usr, opt, tmp 等）:

```
![image.png](https://raw.githubusercontent.com/ScuDays/MyImg/master/202412261126388.png)

### rpm包
##### 概念
RedHat，Fedora，Centos等Linux发行版的软件安装包。

## dpkg工具

### 概念

底层的包管理工具，主要用于对已下载到本地和已经安装的deb包进行管理

### 常用命令

```c
安装软件:dpkg -i xxxx.deb
```

```c
查看安装目录:dpkg -L xxxx
```

```
显示版本:dpkg -l xxxx
```

```
详细信息:dpkg -s xxxx
```

```
罗列内容:dpkg -c xxxx.deb
```

```
卸载软件:dpkg -r xxxx
```

## deb包文件结构分析

- DEBIAN目录:

  - control文件:

    - Package:软件名称

    - Version:版本

    - Section:软件类别

    - Priority:对系统的重要性

    - Architecture:支持的硬件平台

    - Maintainer:软件包的维护者
    - Description:对软件的描述

  - preinst文件 : 安装之前执行的shell脚本
  - postinst文件 : 安装之后执行的shell脚本
  - prerm文件：卸载之前执行的shell脚本
  - postrm文件: 卸载之后执行的shell脚本
  - copyright文件:版权声明
  - changlog文件: 修改记录

- 软件具体安装目录:视实际需求
```
soft-name
    |--DEBIAN
    |       |--control
    |       |--postinst
    |       |--postrm
    |       |--preinst
    |       |--prerm
    |       |--copyright
    |
    |--opt
    |   |--files
    |--etc
    |   |--files
    ...
```

![](https://raw.githubusercontent.com/ScuDays/MyImg/master/202412261126388.png)

## 如何构建一个helloworld的deb包

最好使用一个脚本，自动导入需要执行的程序以及一些信息

## apt命令和apt-get命令
- apt是新版的包管理工具
- 解决apt-get命令过于分散的问题
- apt默认属性对用户友好(进度条、提示升级包数)
