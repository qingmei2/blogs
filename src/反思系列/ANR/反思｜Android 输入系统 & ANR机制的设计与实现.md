# 反思｜Android 输入系统 & ANR机制的设计与实现

## 概述

对于`Android`开发者而言，`ANR`是一个老生常谈的问题，站在面试者的角度，似乎说出 **「不要在主线程做耗时操作」** 就算合格了。

但是，`ANR`机制到底是什么，其背后的原理究竟如何，为什么要设计出这样的机制？这些问题时时刻刻会萦绕脑海，而想搞清楚这些，就不得不提到`Android`自身的 **输入系统** （`Input System`）。

`Android`自身的 **输入系统** 又是什么？一言以蔽之，任何与`Android`设备的交互都需要通过其进行管理和分发；其中最靠近上层，并且开发者最熟悉的一个小环节就是`View`的 **事件分发** 流程。

> 对于事件分发机制的分析，可以参考笔者的这两篇文章：
> * [Android 事件分发机制的设计与实现](https://juejin.im/post/6844903926446161927)
> * [Android 事件拦截机制的设计与实现](https://juejin.im/post/6844904128397705230)

这样看来，**输入系统** 本身确实是一个非常庞大复杂的命题，并且，越靠近底层细节，越容易有一种 **只见树木不见树林** 之感，反复几次，直至迷失在细节代码的较真中，一次学习的努力尝试付诸东流。

因此，**控制住原理分析的粒度，在宏观的角度，系统地了解输入系统本身的设计理念，并引申到实际开发中的`ANR`现象的原理和解决思路** ，是一个非常不错的理论与实践相结合的学习方式，这也正是笔者写作本文的初衷。

本文篇幅较长，整体思维导图如下：

// TODO

## 一、自顶向下探索

谈到`Android`系统本身，首先，必须将 **应用进程** 和 **系统进程** 有一个清晰的认知，前者一般代表开发者依托`Android`平台本身创造开发的应用；后者则代表 `Android`系统自身创建的核心进程。

这里我们抛开 **应用进程** ，先将视线转向 **系统进程**，因为 **输入系统** 本身是由后者初始化和管理调度的。

`Android`系统在启动的时候,会初始化`zygote`进程和由`zygote`进程`fork`出来的`SystemServer`进程；作为 **系统进程** 之一，`SystemServer`进程会提供一系列的系统服务，而接下来要讲到的`InputManagerService`也正是由 `SystemServer` 提供的。

在`SystemServer`的初始化过程中，`InputManagerService`(下称`IMS`)和`WindowManagerService`(下称`WMS`)被创建出来；其中`WMS`本身的创建依赖`IMS`对象的注入：

```Java
// SystemServer.java
private void startOtherServices() {
 // ...
 InputManagerService inputManager = new InputManagerService(context);
 // inputManager作为WindowManagerService的构造参数
 WindowManagerService wm = WindowManagerService.main(context,inputManager, ...);
}
```

在 **输入系统** 中，`WMS`非常重要，其负责管理`IMS`、`Window`与`ActivityManager`之间的通信，这里点到为止，后文再进行补充，我们先来看`IMS`。

顾名思义，`IMS`服务的作用就是负责输入模块在`Java`层级的初始化，并通过`JNI`调用，在`Native`层进行更下层输入子系统相关功能的创建和预处理。

在`JNI`的调用过程中，`IMS`创建了`NativeInputManager`实例，`NativeInputManager`则在初始化流程中又创建了`EventHub`和`InputManager`:

```Cpp
NativeInputManager::NativeInputManager(jobject contextObj, jobject serviceObj, const sp<Looper>& looper) : mLooper(looper), mInteractive(true) {
    // ...
    // 创建一个EventHub对象
    sp<EventHub> eventHub = new EventHub();
    // 创建一个InputManager对象
    mInputManager = new InputManager(eventHub, this, this);
}
```

此时我们已经处于`Native`层级。读者需要注意，对于整个`Native`层级而言，其向下负责与`Linux`的设备节点中获取输入，向上则与靠近用户的`Java`层级相通信，可以说是非常重要。而在该层级中，`EventHub`和`InputManager`又是最核心的两个角色。

这两个角色的职责又是什么呢？首先来说`EventHub`，它是底层 **输入子系统** 中的核心类，负责从物理输入设备中不断读取事件（`Event`)，然后交给`InputManager`,后者内部封装了`InputReader`和`InputDispatcher`，用来从`EventHub`中读取事件和分发事件：

```Cpp
InputManager::InputManager(...) {
    mDispatcher = new InputDispatcher(dispatcherPolicy);
    mReader = new InputReader(eventHub, readerPolicy, mDispatcher);
    initialize();
}
```

简单来看，`EventHub`建立了`Linux`与输入设备之间的通信，`InputManager`中的`InputReader`和`InputDispatcher`负责了输入事件的读取和分发，在 **输入系统** 中，两者的确非常重要。

这里借用网上的图对此进行一个简单的概括：

![](https://raw.githubusercontent.com/qingmei2/qingmei2-blogs-art/master/qingmei2-blogs-art/blogs/2020/QQ20200816-135006%402x.w0j0n7b4e5.png)

## 二、EventHub 与 epoll 机制

对于`EventHub`的具体实现，绝大多数`App`开发者也许并不需要去花太多时间深入——简单了解其职责，然后一笔带过似乎是笔划算的买卖。

但是在`EventHub`的实现细节中笔者发现，其对`epoll`机制的利用是一个非常经典的学习案例，因此，花时间稍微深入了解也绝对是一举两得。

上文说到，`EventHub`建立了`Linux`与输入设备之间的通信，其实这种描述是不准确的，那么，`EventHub`是为了解决什么问题而设计的呢，其具体又是如何实现的？

### 1、多输入设备与输入子系统

我们知道，`Android`设备可以同时连接多个输入设备，比如 **屏幕** 、 **键盘** 、 **鼠标** 等等，用户在任意设备上的输入都会产生一个中断，经由`Linux`内核的中断处理及设备驱动转换成一个`Event`，最终交给用户空间的应用程序进行处理。

`Linux`内核提供了一个便于将不同设备不同数据接口统一转换的抽象层，只要底层输入设备驱动程序按照这层抽象接口实现，应用就可以通过统一接口访问所有输入设备，这便是`Linux`内核的 **输入子系统**。

那么 **输入子系统** 如何是针对接收到的`Event`进行的处理呢？这就不得不提到`EventHub`了，它是底层`Event`处理的枢纽，其利用了`epoll`机制，不断接收到输入事件`Event`，然后将其向上层的`InputReader`传递。

### 2、什么是epoll机制

这是常见于面试`Handler`相关知识点时的一道进阶题，变种问法是：「既然`Handler`中的`Looper`中通过一个死循环不断轮询，为什么程序没有因为无限死循环导致崩溃或者`ANR`?」

读者应该知道，`Handler`简单的利用了`epoll`机制，做到了消息队列的阻塞和唤醒。关于`epoll`机制，这里有一篇非常经典的解释，不了解其设计理念的读者 **有必要** 了解一下：

> [知乎：epoll或者kqueue的原理是什么?](https://www.zhihu.com/question/20122137/answer/14049112)

参考上文，让我们对`epoll`机制进行一个简单的总结:

> `epoll`可以理解为`event poll`，不同于忙轮询和无差别轮询，在 **多个输入流** 的情况下，`epoll`只会把哪个流发生了怎样的I/O事件通知我们。此时我们对这些流的操作都是有意义的。

`EventHub`中使用`epoll`的恰到好处——多个物理输入设备对应了多个不同的输入流，通过`epoll`机制，在`EventHub`初始化时，分别创建`mEpollFd`和`mINotifyFd`；前者用于监听设备节点是否有设备文件的增删，后者用于监听是否有可读事件，创建管道，让`InputReader`来读取事件：

![](https://raw.githubusercontent.com/qingmei2/qingmei2-blogs-art/master/qingmei2-blogs-art/blogs/2020/image2.wn7m6kwmz8.png)
