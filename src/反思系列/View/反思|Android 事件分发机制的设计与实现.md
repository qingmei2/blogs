# 反思|Android 事件分发机制的设计与实现

> **反思** 系列博客是我的一种新学习方式的尝试，该系列起源和目录请参考 [这里](https://github.com/qingmei2/android-programming-profile/blob/master/src/%E5%8F%8D%E6%80%9D%E7%B3%BB%E5%88%97/%E5%8F%8D%E6%80%9D%7C%E7%B3%BB%E5%88%97%E7%9B%AE%E5%BD%95.md) 。

## 概述

`Android`体系本身非常宏大，源码中值得思考和借鉴之处众多。以事件分发完整流程为例，其整个流程涉及到了 **系统启动流程**（`SystemServer`）、**输入管理**(`InputManager`)、**系统服务和UI的通信**（`ViewRootImpl` + `Window` + `WindowManagerService`）、**输入事件分发** 等等一系列复杂的逻辑。

对于 **输入事件分发** 而言，不可否认非常重要，但`Android`系统整体的事件分发流程也是成为一名优秀`Android`工作者需要去了解的，本文笔者将针对`Android` **事件分发** 的整体机制和设计思路进行描述，其整体结构如下图：

![](https://raw.githubusercontent.com/qingmei2/qingmei2-blogs-art/master/android/core/event_dispatcher/image.0018xsd5yv1ynl.png)

## 整体思路

### 1.架构设计

`Android`系统中将输入事件定义为`InputEvent`，而`InputEvent`根据输入事件的类型又分为了`KeyEvent`和`MotionEvent`，前者对应键盘事件，后者则对应屏幕触摸事件，这些事件统一由系统输入管理器`InputManager`进行分发。

在系统启动的时候，`SystemServer`会启动窗口管理服务`WindowManagerService`，`WindowManagerService`在启动的时候就会通过启动系统输入管理器`InputManager`来总负责监控键盘消息。

`InputManager`负责从硬件接收输入事件，并将事件分发给当前激活的窗口（`Window`）处理，这里我们将前者理解为系统层级的 **服务**，将后者理解为应用层级的 **UI**, 因此需要有一个中介负责 **服务** 和 **UI** 之间的通信，于是`ViewRootImpl`类应运而生。

### 2.建立通信

`ActivityThread`负责控制`Activity`的启动过程，在`performLaunchActivity()`流程中，`ActivityThread`会针对`Activity`创建对应的`PhoneWindow`和`DecorView`实例，而之后的`handleResumeActivity()`流程中则会将`PhoneWindow`（ **应用** ）和系统硬件层级的`InputManagerService`( **系统服务** )通信建立对应的连接，保证UI可见并能够对输入事件进行正确的分发，这之后`Activity`就会成为可见的。

如何在应用程序和输入事件的系统服务之间建立通信？`Android`中`Window`和`InputManagerService`之间的通信实际上使用的`InputChannel`,`InputChannel`是一个`pipe`，底层实际是通过`socket`进行通信：

![](https://raw.githubusercontent.com/qingmei2/qingmei2-blogs-art/master/android/core/event_dispatcher/image.256e9cnvm76.png)

在`ActivityThread`的`handleResumeActivity()`流程中, 会通过`WindowManagerImpl.addView()`为当前的`Window`创建一个`ViewRootImpl`实例，当`InputManager`监控到硬件层级的输入事件时，会通知`ViewRootImpl`对输入事件进行底层的事件分发。

### 3.事件分发

与`View`的 [布局流程](https://github.com/qingmei2/android-programming-profile/blob/master/src/反思系列/View/反思%7CAndroid%20View机制设计与实现：布局流程.md) 和 [测量流程](https://github.com/qingmei2/android-programming-profile/blob/master/src/%E5%8F%8D%E6%80%9D%E7%B3%BB%E5%88%97/View/%E5%8F%8D%E6%80%9D%7CAndroid%20View%E6%9C%BA%E5%88%B6%E8%AE%BE%E8%AE%A1%E4%B8%8E%E5%AE%9E%E7%8E%B0%EF%BC%9A%E6%B5%8B%E9%87%8F%E6%B5%81%E7%A8%8B.md) 相同，`Android`事件分发处理机制也使用了 **递归** 的思想，因为一个事件最多只有一个消费者，所以通过责任链的方式将事件自顶向下进行传递，找到事件的消费者（这里是指一个`View`）之后，再自底向上返回结果。

读到这里，读者应该觉得非常熟悉了，但实际上这里描述的事件分发流程只是 **事件分发流程** 的一部分，即UI层级的事件分发——读者需要理解，`ViewRootImpl`从`InputManager`获取到输入事件时，会针对输入事件通过一个复杂的 **责任链** 进行底层的递归，将不同类型的输入事件进行不同策略的分发（比如 **屏幕触摸事件** 和 **键盘输入事件** ），而只有部分符合条件的 **屏幕触摸事件** 最终才有可能进入到UI层级的事件分发。

为了方便对事件分发的整体流程进行描述，本文将其区分为两个小节：**应用全局的事件分发** 和 **UI层级的事件分发** ——需要重申的是，这两个小节虽然被分开讲解，但其本质仍然只是一个完整的 **事件分发的责任链**，后者只是前者的一小部分而已。
