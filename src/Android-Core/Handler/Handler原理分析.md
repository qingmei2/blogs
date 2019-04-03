# Handler原理分析

`Handler`的原理分析这个标题，很多文章都写过，最近认真将源码逐行一字一句研究，特此自己也总结一遍。

首先是`Handler`整个Android消息机制的简单概括：


分三部分对消息机制的整个流程进行阐述：

* `Handler`的创建，包括`Looper`、`MessageQueue`的创建；
* `Handler`发送消息，`Message`是如何进入消息队列`MessageQueue`的（入列）；
* `Looper`轮询消息，`Message`出列，`Handler`处理消息。

## 一、Handler创建流程分析

#### 1.Handler如何被创建的
```Java
// 最简单的创建方式
public Handler() {
    this(null, false);
}

// ....还有很多种方式，但这些方式最终都执行这个构造方法
public Handler(Callback callback, boolean async) {
  // 1.检查内存泄漏
  if (FIND_POTENTIAL_LEAKS) {
      final Class<? extends Handler> klass = getClass();
      if ((klass.isAnonymousClass() || klass.isMemberClass() || klass.isLocalClass()) &&
              (klass.getModifiers() & Modifier.STATIC) == 0) {
          Log.w(TAG, "The following Handler class should be static or leaks might occur: " +
              klass.getCanonicalName());
      }
  }

  // 2.通过Looper.myLooper()获取当前线程的Looper对象
  mLooper = Looper.myLooper();
  // 3.如果Looper为空，抛出异常
  if (mLooper == null) {
      throw new RuntimeException(
          "Can't create handler inside thread " + Thread.currentThread()
                  + " that has not called Looper.prepare()");
  }
  mQueue = mLooper.mQueue;
  mCallback = callback;
  mAsynchronous = async;
}
```

首先，如何避免`Handler`的内存泄漏是一个非常常见的面试题，其实`Handler`的源码中已经将答案非常清晰告知给了开发者,即让`Handler`的导出类保证为`static`的，如果需要，将`Context`作为弱引用的依赖注入进来。

同时，在`Handler`创建的同时，会尝试获取当前线程唯一的`Looper`对象：

```Java
public final class Looper {

  static final ThreadLocal<Looper> sThreadLocal = new ThreadLocal<Looper>();

  public static @Nullable Looper myLooper() {
      return sThreadLocal.get();
  }
}
```

关于`ThreadLocal`，我在[上一篇文章](https://github.com/qingmei2/android-programming-profile/blob/master/src/Android-Core/ThreadLocal/ThreadLocal%E6%BA%90%E7%A0%81%E8%A7%A3%E6%9E%90.md)中已经进行了分析，现在我们知道了`ThreadLocal`保证了当前线程内有且仅有唯一的一个`Looper`。

#### 2.Looper是如何保证线程单例的

那就是需要调用`Looper.prepare()`方法：

```java
public final class Looper {

   public static void prepare() {
       prepare(true);
   }

   private static void prepare(boolean quitAllowed) {
       if (sThreadLocal.get() != null) {
           throw new RuntimeException("Only one Looper may be created per thread");
       }
       sThreadLocal.set(new Looper(quitAllowed));
   }

   private Looper(boolean quitAllowed) {
        mQueue = new MessageQueue(quitAllowed);
        mThread = Thread.currentThread();
    }
 }
```

这也就说明了，为什么当前线程没有`Looper`的实例时，会抛出一个异常并提示开发者需要调用`Looper.prepare()`方法了。

也正如上述代码片段所描述的，如果当前线程已经有了`Looper`的实例，也会抛出一个异常，提示用户每个线程只能有一个`Looper`（`throw new RuntimeException("Only one Looper may be created per thread");`）。

此外，在`Looper`实例化的同时，也创建了对应的`MessageQueue`，这也就说明，一个线程有且仅有一个`Looper`，也仅有一个`MessageQueue`。

## 二、发送消息流程分析

#### 1.sendMessage()分析

`sendMessage()`流程如下：

```java
// 1.发送即时消息
public final boolean sendMessage(Message msg)
{
    return sendMessageDelayed(msg, 0);
}

// 2.实际上是发射一个延时为0的Message
public final boolean sendMessageDelayed(Message msg, long delayMillis)
{
   if (delayMillis < 0) {
       delayMillis = 0;
   }
   return sendMessageAtTime(msg, SystemClock.uptimeMillis() + delayMillis);
}

// 3.将消息和延时的时间进行入列（消息队列）
public boolean sendMessageAtTime(Message msg, long uptimeMillis) {
    MessageQueue queue = mQueue;
    if (queue == null) {
        RuntimeException e = new RuntimeException(
                this + " sendMessageAtTime() called with no mQueue");
        Log.w("Looper", e.getMessage(), e);
        return false;
    }
    return enqueueMessage(queue, msg, uptimeMillis);
}

// 4.内部实际上还是执行了MessageQueue的enqueueMessage()方法
private boolean enqueueMessage(MessageQueue queue, Message msg, long uptimeMillis) {
    msg.target = this;
    if (mAsynchronous) {
        msg.setAsynchronous(true);
    }
    return queue.enqueueMessage(msg, uptimeMillis);
}
```

注意第四步实际上将`Handler`对象最为target，附着在了`Message`之上；接下来看`MessageQueue`类内部是如何对`Message`进行入列的。

#### 2.MessageQueue消息入列

```Java
boolean enqueueMessage(Message msg, long when) {
    //... 省略部分代码
    synchronized (this) {
        msg.markInUse();
        msg.when = when;
        // 获得链表头的Message
        Message p = mMessages;
        boolean needWake;
        if (p == null || when == 0 || when < p.when) {
            // 若有以下情景之一，将Message置于链表头
            // 1.头部Message为空，链表为空
            // 2.消息为即时Message
            // 3.头部Message的时间戳大于最新Message的时间戳
            msg.next = p;
            mMessages = msg;
            needWake = mBlocked;
        } else {
            // 反之，将Message插入到链表对应的位置
            Message prev;
            // for循环就是找到合适的位置，并将最新的Message插入链表
            for (;;) {
                prev = p;
                p = p.next;
                if (p == null || when < p.when) {
                    break;
                }
            }
            msg.next = p; // invariant: p == prev.next
            prev.next = msg;
        }
        if (needWake) {
            nativeWake(mPtr);
        }
    }
    return true;
}
```

`MessageQueue`的数据结构本身是一个**单向链表**。

## 三、接收消息分析

当`Handler`创建好后，若在此之前调用了`Looper.prepare()`初始化`Looper`，还需要调用`Looper.loop()`开始该线程内的消息轮询。

#### 1.Looper.loop()

```java
public static void loop() {
    // ...省略部分代码
    // 1. 获取Looper对象
    final Looper me = myLooper();
    if (me == null) {
        throw new RuntimeException("No Looper; Looper.prepare() wasn't called on this thread.");
    }
    // 2.获取messageQueue
    final MessageQueue queue = me.mQueue;
    // 3. 轮询消息，这里是一个死循环
    for (;;) {
        // 4.从消息队列中取出消息，若消息队列为空，则阻塞线程
        Message msg = queue.next();
        if (msg == null) {
            return;
        }

        // 5.派发消息到对应的Handler
        msg.target.dispatchMessage(msg);
        // ...
    }
}
```
比较简单，需要注意的一点是`MessageQueue.next()`是一个可能会阻塞线程的方法，当有消息时会轮询处理消息，但如果消息队列中没有消息，则会阻塞线程。

#### 2.MessageQueue.next()

```Java
private native void nativePollOnce(long ptr, int timeoutMillis);


Message next() {

    // ...省略部分代码
    int nextPollTimeoutMillis = 0;

    for (;;) {
      // ...

    // native方法
    nativePollOnce(ptr, nextPollTimeoutMillis);

    synchronized (this) {

        final long now = SystemClock.uptimeMillis();
        Message prevMsg = null;
        Message msg = mMessages;

        // 从消息队列中取出消息
        if (msg != null) {
            // 当时间小于message的时间戳时，获取时间差
            if (now < msg.when) {
                // 该值将会导致在下次循环中阻塞对应时间
                nextPollTimeoutMillis = (int) Math.min(msg.when - now, Integer.MAX_VALUE);
            } else {
                // 取出消息并返回
                mBlocked = false;
                if (prevMsg != null) {
                    prevMsg.next = msg.next;
                } else {
                    mMessages = msg.next;
                }
                msg.next = null;
                msg.markInUse();
                return msg;
            }
        }
        // ...
   }
}
```

注意代码片段最上方的native方法——循环体内首先调用`nativePollOnce(ptr, nextPollTimeoutMillis)`，这是一个native方法，实际作用就是通过Native层的`MessageQueue`阻塞`nextPollTimeoutMillis`毫秒的时间:

* 1.如果nextPollTimeoutMillis=-1，一直阻塞不会超时。
* 2.如果nextPollTimeoutMillis=0，不会阻塞，立即返回。
* 3.如果nextPollTimeoutMillis>0，最长阻塞nextPollTimeoutMillis毫秒(超时)，如果期间有程序唤醒会立即返回。

搞清楚这一点，其它就都好理解了。

### 3.最终将消息发送给Handler

正如上文所说的，`msg.target.dispatchMessage(msg)`实际上就是调用`Handler.dispatchMessage(msg)`,内部最终也是执行了`Handler.handleMessage()`回调：

```Java
public void dispatchMessage(Message msg) {
    if (msg.callback != null) {
        handleCallback(msg);
    } else {
        if (mCallback != null) {
            if (mCallback.handleMessage(msg)) {
                return;
            }
        }
        // 如果消息没有定义callBack，或者不是通过
        // Handler(Callback)的方式实例化Handler,
        // 最终会走到这里
        handleMessage(msg);
    }
}
```

## 参考&感谢

* 《Android开发艺术探索》
* [深入理解MessageQueue](https://blog.csdn.net/kisty_yao/article/details/71191175)
* [Android Handler：手把手带你深入分析 Handler机制源码](https://www.jianshu.com/p/b4d745c7ff7a)

---

## 关于我

Hello，我是[却把清梅嗅](https://github.com/qingmei2)，如果您觉得文章对您有价值，欢迎 ❤️，也欢迎关注我的[博客](https://www.jianshu.com/u/df76f81fe3ff)或者[Github](https://github.com/qingmei2)。

如果您觉得文章还差了那么点东西，也请通过**关注**督促我写出更好的文章——万一哪天我进步了呢？

* [我的Android学习体系](https://github.com/qingmei2/android-programming-profile)
* [关于文章纠错](https://github.com/qingmei2/Programming-life/blob/master/error_collection.md)
* [关于知识付费](https://github.com/qingmei2/Programming-life/blob/master/appreciation.md)
