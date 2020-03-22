---
title: Android多屏幕专题--开篇
date: 2020-03-21 21:58:57
tags:
- Android
- AMS
- WMS
- Input
---

# Android多屏幕

​	Android从4.4开始其实就已经有了(不确定最初是哪个版本)，在Android6.0的时候就有Presentation作为异显开发，而到AndroidQ，就已经发现有双Home(两个launcher，可以在两个显示屏上打开应用)。而我对这一块一直都比较感兴趣，而且涉及了Android frameworks很多的内容，诸如AMS、WMS、DisplayManagerService、SurfaceFlinger和Input等等。随着嵌入式芯片的性能越来越强，多屏操作也就不再吃力，Android也越来越接近像windows那样强大的界面操作能力。

## 分析步骤

​	由于多屏幕异显部分设计的子系统繁多，我们将逐一分块进行分析，大概步骤如下:

​	1.从双home场景进行分析，了解两个Home是如何启动与显示。

​	2.了解Presentation的原理，分析其是如何显示到其他的屏幕上。

​	3.了解AMS是如何管理多个屏幕的，如何组织或者启动一个Activity到另一个屏幕上。

​	4.了解WMS是如何管理多个屏幕的windows。

​	5.了解SurfaceFlinger是如何合成和显示不同屏幕的视图。

​	6.了解InputFlinger的不同屏幕的Viewport更新机制。

​	……