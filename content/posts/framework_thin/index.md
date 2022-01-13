---
title: "APP瘦身 - 之framework"
date: 2017-11-05T17:19:51+08:00
draft: true
---

在大型项目的开发中，我们常常会对一些功能比较固定的模块进行封装打包成静态库，以提供给他人使用。但在一般的打包中，得到的framework体积都比较大，这样会导致最后开发出来的APP，体积会过大，用户从App Store下载时，消耗大量的时间和流量，最终影响用户体验。

当我们决定要将一个工程打包成framework供他人使用时，通常支持的指令集有i386、x86_64、armv7、arm64，其中armv7和arm64为供真机的指令集，i386和x86_64电脑端的指令集。往往我们打出来的包，是可以供支持电脑端和真机使用的，通常这样的framework体积会比较大。如果单独，将包分成真机部分和电脑虚拟机部分，会节省出很大的一笔空间。

lipo 是一个在 Mac OS X 中处理通用程序（Universal Binaries）的工具。使用lipo，我们可以将framework中的静态库，进行分割。
以下是一些常用指令：

* 查看静态库支持的CPU架构

```
lipo -info libname.a(或者libname.framework/libname)
```

* 合并静态库

```
lipo -create 静态库存放路径1  静态库存放路径2 ...  -output 整合后存放的路径
lipo  -create  libname-armv7.a   libname-armv7s.a   libname-i386.a  -output  libname.a
```
framework 合并(例util.framework)

```
lipo -create  arm7/util.framework/util  i386/util.framework/util   -output   util.framework   
```

* 静态库拆分

```
 lipo 静态库源文件路径 -thin CPU架构名称 -output 拆分后文件存放路径
```
架构名为armv7/armv7s/arm64等，与lipo -info 输出的架构名一致

```
lipo  libname.a  -thin  armv7  -output  libname-armv7.a
```

对framework进行瘦身的方案可以如下：

```
//找到framework中的静态文件staticLib
//分别打包出各个平台的库
lipo staticLib -thin armv7 -output staticLib_armv7
lipo staticLib -thin arm64 -output staticLib_arm64
lipo staticLib -thin i386 -output staticLib_i386
lipo staticLib -thin x86_64 -output staticLib_x86_64

//分别组合出真机平台，和电脑虚拟机平台的包
lipo -create staticLib_armv7 staticLib_arm64   -output   staticLib_iphone
lipo -create staticLib_i386 staticLib_x86_64   -output   staticLib_iphonesimulate

```

关键代码可以用Python进行包装，方便以后调用。

```
import os

os.system("lipo staticLib -thin armv7 -output staticLib_armv7")

...
```

[参考文章](http://www.jianshu.com/p/e590f041c5f6)
