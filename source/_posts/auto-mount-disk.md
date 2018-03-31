---
title: 自动挂载硬盘
author: 肖哥
avatar: /yxiao.jpg
authorLink: https://blog.imyxiao.com
authorAbout: https://blog.imyxiao.com/about
authorDesc: Java Developer
date: 2017-08-22 21:29:51
categories: linux
tags:
  - mount
  - disk
  - deepin
description: 我的系统（deepin）装在了固态硬盘上，并且只分配了２０G空间．并不是很够用，所以很多数据、程序放在了之前的机械硬盘上，但是硬盘不是自己挂载的，需要开机后点一下才会挂载，挺麻烦。所以需要开机自动挂载硬盘。
---
我的系统（deepin）装在了固态硬盘上，并且只分配了２０G空间．并不是很够用，所以很多数据、程序放在了之前的机械硬盘上，但是硬盘不是自己挂载的，需要开机后点一下才会挂载，挺麻烦。所以需要开机自动挂载硬盘。

### 查询NTFS磁盘的UUID

```shell
sudo blkid
```
查询出所以磁盘的信息，其中一条如下：
```java
/dev/sda5: LABEL="work" UUID="0005F366000EF1F7" TYPE="ntfs" PARTUUID="e6f99ced-05"
```

### 编辑/etc/fstab
```shell
sudo vim /etc/fstab
```
将下面这段添加到末尾即可，`UUID`为上一步查询所得，`/media/yxiao/work`是挂载点
```java
UUID=0005F366000EF1F7    /media/yxiao/work    ntfs    defaults    0    0
```
保存退出即可。下次开机会自动挂载。
