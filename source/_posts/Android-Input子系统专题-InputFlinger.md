---
title: Android Input子系统专题-InputFlinger(一)
date: 2020-03-15 21:39:40
tags:
- Android
- Input
---

# InputFlinger的启动

​	InputFlinger在Android(至少在AndroiP及以后的版本)中作为一个服务而存在，如果有学习过Android的服务启动，那么对InputFlinger的启动也将会十分熟悉，在本文将简单地介绍一下它的启动。

​	InputFlinger的源码在`frameworks/native/services/inputflinger`目录下，在该目录下有一个`host`目录，这里面就是InputFlinger的服务程序，而与host目录同级下的源码则是InputFlinger的核心代码--libinputflinger。

​	在`host`目录下有一个`inputflinger.rc`文件：

```
service inputflinger /system/bin/inputflinger
    class main
    user system
    group input wakelock
#    onrestart restart zygote
```

​	InputFlinger属于main组别的服务，group是input和wakelock，由于需要读取`/dev/input/*`下的设备节点，因此需要input group，而且部分设备还具备唤醒系统，因此还有wakelock的group。

​	最后一个`onrestart restart zygote`，说明之前的版本inputflinger如果挂掉，那么Android将会重启，但AndroidP后就不需要这么做了，因为当inputflinger挂掉后，init会将它重启。而其他地方估计是没有其他的服务去引用它，其相对独立。这个可在后续研究一下。

