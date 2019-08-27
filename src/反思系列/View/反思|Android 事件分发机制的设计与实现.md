# 反思|Android 事件分发机制的设计与实现

## 参考

* Android 源码
* [Android应用程序输入事件分发和处理机制](https://www.kancloud.cn/alex_wsc/androids/472164)
* [View InputEvent事件投递源码分析 1-4](https://www.jianshu.com/p/b7f33f46d33c)
* [Android中的ViewRootImpl类源码解析](https://blog.csdn.net/qianhaifeng2012/article/details/51737370)


> **反思** 系列博客是我的一种新学习方式的尝试，该系列起源和目录请参考 [这里](https://github.com/qingmei2/android-programming-profile/blob/master/src/%E5%8F%8D%E6%80%9D%E7%B3%BB%E5%88%97/%E5%8F%8D%E6%80%9D%7C%E7%B3%BB%E5%88%97%E7%9B%AE%E5%BD%95.md) 。

## 概述

`Android`体系本身非常宏大，源码中值得思考和借鉴之处众多。以事件分发完整流程为例，其整个流程涉及到了 **系统启动流程**（`SystemServer`）、**输入管理**(`InputManager`)、**系统服务和UI的通信**（`ViewRootImpl` + `Window` + `WindowManagerService`）、**输入事件分发** 等等一系列复杂的逻辑。

对于 **输入事件分发** 而言，不可否认非常重要，但`Android`系统整体的事件分发流程也是成为一名优秀`Android`工作者需要去了解的，本文笔者将针对`Android` **事件分发** 的整体机制和设计思路进行描述，其整体结构如下图：

![](https://raw.githubusercontent.com/qingmei2/qingmei2-blogs-art/master/android/core/event_dispatcher/image.0018xsd5yv1ynl.png)

## 整体思路

### 1.架构设计

`Android`系统中将输入事件定义为`InputEvent`，而`InputEvent`根据输入事件的类型又分为了`KeyEvent`和`MotionEvent`，前者对应键盘事件，后者则对应屏幕触摸事件，这些事件统一由系统输入管理器`InputManager`进行分发。

在系统启动的时候，`SystemServer`会启动窗口管理服务`WindowManagerService`，`WindowManagerService`在启动的时候就会通过启动系统输入管理器`InputManager`来总负责监控键盘消息。

`InputManager`负责从硬件接收输入事件，并将事件分发给当前激活的窗口（`Window`）处理，这里我们将前者理解为 **系统服务**，将后者理解为应用层级的 **UI**, 因此需要有一个中介负责 **服务** 和 **UI** 之间的通信，于是`ViewRootImpl`类应运而生。

### 2.建立通信

`ActivityThread`负责控制`Activity`的启动过程，在`performLaunchActivity()`流程中，`ActivityThread`会针对`Activity`创建对应的`PhoneWindow`和`DecorView`实例，而之后的`handleResumeActivity()`流程中则会将`PhoneWindow`（ **应用** ）和系统硬件层级的`InputManagerService`( **系统服务** )通信建立对应的连接，保证UI可见并能够对输入事件进行正确的分发，这之后`Activity`就会成为可见的。

如何在应用程序和系统服务之间建立通信？`Android`中`Window`和`InputManagerService`之间的通信实际上使用的`InputChannel`,`InputChannel`是一个`pipe`，底层实际是通过`socket`进行通信：

![](https://raw.githubusercontent.com/qingmei2/qingmei2-blogs-art/master/android/core/event_dispatcher/image.256e9cnvm76.png)

在`ActivityThread`的`handleResumeActivity()`流程中, 会通过`WindowManagerImpl.addView()`为当前的`Window`创建一个`ViewRootImpl`实例，当`InputManager`监控到硬件层级的输入事件时，会通知`ViewRootImpl`对输入事件进行底层的事件分发。

### 3.事件分发

与`View`的 [布局流程](https://github.com/qingmei2/android-programming-profile/blob/master/src/反思系列/View/反思%7CAndroid%20View机制设计与实现：布局流程.md) 和 [测量流程](https://github.com/qingmei2/android-programming-profile/blob/master/src/%E5%8F%8D%E6%80%9D%E7%B3%BB%E5%88%97/View/%E5%8F%8D%E6%80%9D%7CAndroid%20View%E6%9C%BA%E5%88%B6%E8%AE%BE%E8%AE%A1%E4%B8%8E%E5%AE%9E%E7%8E%B0%EF%BC%9A%E6%B5%8B%E9%87%8F%E6%B5%81%E7%A8%8B.md) 相同，`Android`事件分发处理机制也使用了 **递归** 的思想，因为一个事件最多只有一个消费者，所以通过责任链的方式将事件自顶向下进行传递，找到事件的消费者（这里是指一个`View`）之后，再自底向上返回结果。

读到这里，读者应该觉得非常熟悉了，但实际上这里描述的事件分发流程只是 *事件分发流程* 整体的一部分，即 *UI层级事件分发* ——读者需要理解，`ViewRootImpl`从`InputManager`获取到输入事件时，会针对输入事件通过一个复杂的 **责任链** 进行底层的递归，将不同类型的输入事件进行不同策略的分发（比如 **屏幕触摸事件** 和 **键盘输入事件** ），而只有部分符合条件的 **屏幕触摸事件** 最终才有可能进入到UI层级的事件分发：

![](https://raw.githubusercontent.com/qingmei2/qingmei2-blogs-art/master/android/core/event_dispatcher/image.nn28mi8lj9m.png)

为了方便理解，本文使用以下两个词汇对上文两个斜体词汇进行描述：**应用整体的事件分发** 和 **UI层级的事件分发** ——需要重申的是，这两个词汇虽然会被分开讲解，但其本质仍然属于一个完整 **事件分发的责任链**，后者只是前者的一小部分而已。

## 架构设计

### 1.InputEvent：输入事件分类概述

`Android`系统中将输入事件定义为`InputEvent`，而`InputEvent`根据输入事件的类型又分为了`KeyEvent`和`MotionEvent`：

```java
// 输入事件的基类
public abstract class InputEvent implements Parcelable { }

public class KeyEvent extends InputEvent implements Parcelable { }

public final class MotionEvent extends InputEvent implements Parcelable { }
```

`KeyEvent`对应了键盘的输入事件，那么什么是`MotionEvent`？顾名思义，`MotionEvent`就是移动事件，鼠标、笔、手指、轨迹球等相关输入设备的事件都属于`MotionEvent`，本文我们简单地将其视为 **屏幕触摸事件**。

用户的输入种类繁多，由此可见，`Android`输入系统的设计中，将 **输入事件** 抽象为`InputEvent`是有必要的。

### 2.InputManager：系统输入管理器

`Android`系统的设计中，`InputEvent`统一由系统输入管理器`InputManager`进行分发。在这里`InputManager`是`native`层级的一个类，负责与硬件通信并接收输入事件。

那么`InputManager`是如何初始化的呢？这里就要涉及到`Java`层级的`SystemServer`了，我们直到`SystemServer`进程中包含着各种各样的系统服务，比如`ActivityManagerService`、`WindowManagerService`等等，`SystemServer`由`zygote`进程启动, 启动过程中对`WindowManagerService`和`InputManagerService`进行了初始化:

```java
public final class SystemServer {

  private void startOtherServices() {
     // 初始化 InputManagerService
     InputManagerService inputManager = new InputManagerService(context);
     // WindowManagerService 持有了 InputManagerService
     WindowManagerService wm = WindowManagerService.main(context, inputManager,...);

     inputManager.setWindowManagerCallbacks(wm.getInputMonitor());
     inputManager.start();
  }
}
```

`InputManagerService`的构造器中，通过调用native函数，通知`native`层级初始化`InputManager`:

```java
public class InputManagerService extends IInputManager.Stub {

  public InputManagerService(Context context) {
    // ...通知native层初始化 InputManager
    mPtr = nativeInit(this, mContext, mHandler.getLooper().getQueue());
  }

  // native 函数
  private static native long nativeInit(InputManagerService service, Context context, MessageQueue messageQueue);
}
```

`SystemServer`会启动窗口管理服务`WindowManagerService`，`WindowManagerService`在启动的时候就会通过`InputManagerService`启动系统输入管理器`InputManager`来总负责监控键盘消息。

对于本文而言，`framework`层级相关如`WindowManagerService`（窗口管理服务）、`native`层级的源码、`SystemServer` 亦或者 `Binder`跨进程通信并非重点，读者仅需了解 **系统服务的启动流程** 和 **层级关系** 即可，参考下图：

![](https://raw.githubusercontent.com/qingmei2/qingmei2-blogs-art/master/android/core/event_dispatcher/image.2m7iml9m5qg.png)

### 3.ViewRootImpl：窗口服务与窗口的纽带

`InputManager`并将事件分发给当前激活的窗口（`Window`）处理，这里我们将前者理解为系统层级的 **（窗口）服务**，将后者理解为应用层级的 **窗口**, 因此需要有一个中介负责 **服务** 和 **窗口** 之间的通信，于是`ViewRootImpl`类应运而生。

`ViewRootImpl`作为链接`WindowManager`和`DecorView`的纽带，同时实现了`ViewParent`接口，`ViewRootImpl`作为整个控件树的根部，它是`View Tree`正常运作的动力所在，控件的测量、布局、绘制以及输入事件的分发都由`ViewRootImpl`控制。

由此可见，`ViewRootImpl`确实很重要，那么`ViewRootImpl`是如何被创建和初始化的，而 **（窗口）服务** 和 **窗口** 之间的通信又是如何建立的呢？

## 建立通信

### 1.ViewRootImpl的创建

既然`Android`系统将 **（窗口）服务** 与 **窗口** 的通信建立交给了`ViewRootImpl`，那么`ViewRootImpl`必然持有了两者的依赖，因此了解`ViewRootImpl`是如何创建的就非常重要。

我们知道，`ActivityThread`负责控制`Activity`的启动过程，在`ActivityThread.performLaunchActivity()`流程中，`ActivityThread`会针对`Activity`创建对应的`PhoneWindow`和`DecorView`实例，而在`ActivityThread.handleResumeActivity()`流程中，`ActivityThread`会将获取当前`Activity`的`WindowManager`，并将`DecorView`和`WindowManager.LayoutParams`(布局参数)作为参数调用`addView()`函数：

```java
// 伪代码
public final class ActivityThread {

  @Override
  public void handleResumeActivity(...){
    //...
    windowManager.addView(decorView, windowManagerLayoutParams);
  }
}
```

`WindowManager.addView()`实际上就是对`ViewRootImpl`进行了初始化，并执行了`setView()`函数：

```java
// 1.WindowManager 的本质实际上是 WindowManagerImpl
public final class WindowManagerImpl implements WindowManager {

   @Override
   public void addView(@NonNull View view, @NonNull ViewGroup.LayoutParams params) {
       // 2.实际上调用了 WindowManagerGlobal.addView()
       WindowManagerGlobal.getInstance().addView(...);
   }
}

public final class WindowManagerGlobal {

   public void addView(...) {
      // 3.初始化 ViewRootImpl，并执行setView()函数
      ViewRootImpl root = new ViewRootImpl(view.getContext(), display);
      root.setView(view, wparams, panelParentView);
   }
}

public final class ViewRootImpl {

  public void setView(View view, WindowManager.LayoutParams attrs, View panelParentView) {
      // 4.该函数就是控测量（measure）、布局(layout)、绘制（draw）的开始
      requestLayout();
      // ...
      // 5.此外还有通过Binder建立通信，这个下文再提
  }
}
```

`Android`系统的`Window`机制并非本文重点，读者可简单理解为`ActivityThread.handleResumeActivity()`流程中最终创建了`ViewRootImpl`，并通过`setView()`函数对`DecorView`开始了绘制流程的三个步骤。

### 2.通信的建立

完成了`ViewRootImpl`的创建之后，如何完成系统输入服务和应用程序进程的链接呢？

`Android`中`Window`和`InputManagerService`之间的通信实际上使用的`InputChannel`,`InputChannel`是一个`pipe`，底层实际是通过`socket`进行通信。在`ViewRootImpl.setView()`过程中，也会同时注册`InputChannel`：

```java
public final class InputChannel implements Parcelable { }
```

上文中，我们提到了`ViewRootImpl.setView()`函数，在该函数的执行过程中，会在`ViewRootImpl`中创建`InputChannel`，`InputChannel`实现了`Parcelable`， 所以它可以通过`Binder`传输。具体是通过`addDisplay()`将当前`window`加入到`WindowManagerService`中管理:

```java
public final class ViewRootImpl {

  public void setView(View view, WindowManager.LayoutParams attrs, View panelParentView) {
      requestLayout();
      // ...
      // 创建InputChannel
      mInputChannel = new InputChannel();
      // 通过Binder在SystemServer进程中完成InputChannel的注册
      mWindowSession.addToDisplay(mWindow, mSeq, mWindowAttributes,
                            getHostVisibility(), mDisplay.getDisplayId(),
                            mAttachInfo.mContentInsets, mAttachInfo.mStableInsets,
                            mAttachInfo.mOutsets, mInputChannel);
  }
}
```

这里涉及到了`WindowManagerService`和`Binder`跨进程通信，读者不需要纠结于详细的细节，只需了解最终在`SystemServer`进程中，`WindowManagerService`根据当前的`Window`创建了`SocketPair`用于跨进程通信，同时并对`App`进程中传过来的`InputChannel`进行了注册，这之后，`ViewRootImpl`里的`InputChannel`就指向了正确的`InputChannel`, 作为`Client`端，其`fd`与`SystemServer`进程中`Server`端的`fd`组成`SocketPair`, 它们就可以双向通信了。

> 对这个流程感兴趣的读者可以参考[这篇文章](https://www.jianshu.com/p/2bff4ecd86c9)

## 应用整体的事件分发

`App`端与服务端建立了双向通信之后，`InputManager`就能够将产生的输入事件从底层硬件分发过来，`Android`提供了`InputEventReceiver`接收这些消息：

```java

```
