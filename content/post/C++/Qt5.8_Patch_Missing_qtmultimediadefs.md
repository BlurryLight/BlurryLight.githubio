---
layout: post
cid: 425
title: "Qt5.8 Patch: Missing qtmultimediadefs.h"
slug: 425
date: 2017-08-22
updated: 2017-08-22
status: publish
author: panda
categories: 
  - cpp
tags: 
---


详情请见：
https://bugreports.qt.io/browse/QTBUG-58432
这是被列在Qt5.8的等级为P1的Bug report,原因是在某次merge中移除了这个`qtmultimediadefs.h`这个文件，**很多头文件的声明被更名到`qtmultimediaglobal.h`，**，并进行了一些其他的修改，但是在打包成安装包的过程中出现了一些问题，加粗部分的变动并没有打进安装包，导致所有引用了qtmultimedia功能的程序均无法编译。


<!--more-->


在Linux中可以利用包管理器解决
这是Debian的软件包信息
https://packages.debian.org/stretch/qtmultimedia5-dev

在Windows中就要麻烦一点了
首先需要找到include的目录
我的目录在`E:\Qt\Qt5.8.0\5.8\mingw53_32\include\QtMultimedia`中，找到`qtmultimediadefs.h`文件。
将
https://codereview.qt-project.org/#/c/184100/2/src/multimedia/qtmultimediadefs.h
中的信息复制到`qtmultimediadefs.h`


或者
升级到Qt5.9，在5.9中已经修复了此Bug
