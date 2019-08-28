### 组装责任链

现在我们理解了，新分发的事件会通过一个`InputStage`的责任链进行整体的事件分发，这意味着，当新的事件到来时，责任链已经组装好了，那么这个责任链是何时进行组装的？

不难得出，对于责任链的组装，最好是在系统服务和`Window`建立通信成功的时候，而上文中也提到了，通信的建立是执行在`ViewRootImpl.setView()`方法中的，因此在`InputChannel`注册成功之后，即可对责任链进行组装：

```java
public final class ViewRootImpl implements ViewParent {

  public void setView(View view, WindowManager.LayoutParams attrs, View panelParentView) {
     // ...
     // 1.开始根布局的绘制流程
     requestLayout();
     // 2.通过Binder建立双端的通信
     res = mWindowSession.addToDisplay(...)
     mInputEventReceiver = new WindowInputEventReceiver(mInputChannel, Looper.myLooper());
     // 3.对责任链进行组装
     mSyntheticInputStage = new SyntheticInputStage();
     InputStage viewPostImeStage = new ViewPostImeInputStage(mSyntheticInputStage);
     InputStage nativePostImeStage = new NativePostImeInputStage(viewPostImeStage,
            "aq:native-post-ime:" + counterSuffix);
     InputStage earlyPostImeStage = new EarlyPostImeInputStage(nativePostImeStage);
     InputStage imeStage = new ImeInputStage(earlyPostImeStage,
            "aq:ime:" + counterSuffix);
     InputStage viewPreImeStage = new ViewPreImeInputStage(imeStage);
     InputStage nativePreImeStage = new NativePreImeInputStage(viewPreImeStage,
            "aq:native-pre-ime:" + counterSuffix);
     mFirstInputStage = nativePreImeStage;
     mFirstPostImeInputStage = earlyPostImeStage;
     // ...
  }
}
```

这说明`ViewRootImpl.setView()`函数非常重要，该函数也正是`ViewRootImpl`本身职责的体现：

* 1.链接`WindowManager`和`DecorView`的纽带，更广一点可以说是`Window`和`View`之间的纽带；
* 2.完成`View`的绘制过程，包括`measure、layout、draw`过程；
* 3.向DecorView分发收到的用户发起的`InputEvent`事件。

最终整体事件分发流程由如下责任链构成：

> SyntheticInputStage --> ViewPostImeStage --> NativePostImeStage --> EarlyPostImeStage --> ImeInputStage --> ViewPreImeInputStage --> NativePreImeInputStage

### 事件分发结果的返回

上文说到，真正从`Native`层的`InputManager`接收输入事件的是`ViewRootImpl`的`WindowInputEventReceiver`对象，既然负责输入事件的分发，自然也负责将事件分发的结果反馈给`Native`层，作为事件分发的结束：

```java
public final class ViewRootImpl {

  final class WindowInputEventReceiver extends InputEventReceiver {
    @Override
     public void onInputEvent(InputEvent event, int displayId) {
         // 【开始】将输入事件加入队列,开始事件分发
         enqueueInputEvent(event, this, 0, true);
     }
  }
}

// ViewRootImpl.WindowInputEventReceiver 是其子类，因此也持有finishInputEvent函数
public abstract class InputEventReceiver {
  private static native void nativeFinishInputEvent(long receiverPtr, int seq, boolean handled);

  public final void finishInputEvent(InputEvent event, boolean handled) {
     //...
     // 【结束】调用native层函数，结束应用层的本次事件分发
     nativeFinishInputEvent(mReceiverPtr, seq, handled);
  }
}
```

### ViewPostImeInputStage：UI层事件分发的起点

上文已经提到，**UI层级的事件分发** 作为 **完整事件分发流程的一部分**，发生在`ViewPostImeInputStage.processPointerEvent`函数中：

```java
final class ViewPostImeInputStage extends InputStage {

  private int processPointerEvent(QueuedInputEvent q) {
    // 让顶层的View开始事件分发
    final MotionEvent event = (MotionEvent)q.mEvent;
    boolean handled = mView.dispatchPointerEvent(event);
    //...
  }
}
```

这个顶层的`View`其实就是`DecorView`（参见上文 *建立通信-ViewRootImpl的创建* 小节），读者知道，`DecorView`实际上就是`Activity`的根布局，它是一个`FrameLayout`。

现在`DecorView`执行了`dispatchPointerEvent(event)`函数，这是不是就意味着开始了`View`的事件分发？

### DecorView的双重身份

`DecorView`作为`View`树的根节点，接收到屏幕触摸事件`MotionEvent`时，通过递归的方式将事件分发给子`View`，这似乎理所当然。但实际设计中，设计者将`DecorView`接收到的事件首先分发给了`Activity`，`Activity`又将事件分发给了其`Window`，最终`Window`才将事件又交回给了`DecorView`，形成了一个小的循环：

```java
// 伪代码
public class DecorView extends FrameLayout {

  // 1.将事件分发给Activity
  @Override
  public boolean dispatchTouchEvent(MotionEvent ev) {
      return window.getActivity().dispatchTouchEvent(ev)
  }

  // 4.执行ViewGroup 的 dispatchTouchEvent
  public boolean superDispatchTouchEvent(MotionEvent event) {
      return super.dispatchTouchEvent(event);
  }
}

// 2.将事件分发给Window
public class Activity {
  public boolean dispatchTouchEvent(MotionEvent ev) {
      return getWindow().superDispatchTouchEvent(ev);
  }
}

// 3.将事件再次分发给DecorView
public class PhoneWindow extends Window {
  @Override
  public boolean superDispatchTouchEvent(MotionEvent event) {
      return mDecor.superDispatchTouchEvent(event);
  }
}
```

对于初次阅读源码的读者来说，这里的设计平淡无奇，似乎说它莫名其妙也不过分，事实上这里是 **面向对象程序设计中灵活运用多态** 这一特征的有力体现——对于`DecorView`而言，它承担了2个职责：

* 1.在接收到输入事件时，`DecorView`不同于其它`View`，它需要先将事件转发给最外层的`Activity`，使得开发者可以通过重写`Activity.onTouchEvent()`函数以达到对当前屏幕触摸事件拦截控制的目的，这里`DecorView`履行了自身的职责；
* 2.从`Window`接收到事件时，作为`View`树的根节点，将事件分发给子`View`，这里`DecorView`履行了一个普通的`View`的职责。

实际上，不只是`DecorView`，接下来 `ViewGroup -> View` 之间的事件分发中也运用到了这个技巧，对于`ViewGroup`的事件分发来说，在 **递流程** 中，其本身是上游的`ViewGroup`，需要将事件分发给下游的子`View`（调用自定义`this.dispatchTouchEvent(event)`）；同时，在 **归流程** 中，其本身也是一个`View`，需要调用`View`自身的方法已决定是否消费该事件（调用`super.dispatchTouchEvent(event)`），并将结果返回上游—— **读者需认真理解这个设计思想，下文中`View`层的事件分发的流程也是基于这个思想而设计的**。

同时，读者应该也已理解，平时我们所说 `ViewGroup -> View` 之间的事件分发也只是 **UI层的事件分发** 的一个环节，而 **UI层的事件分发** 又只是 **应用层完整事件分发** 的一个小环节，更遑论`Native`层和应用层之间的事件分发机制了。

## UI层级事件分发

本小节将 **UI层级事件分发** 归纳为`ViewGroup -> View` 之间的事件分发，
