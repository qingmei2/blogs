# ThreadLocal原理分析

> 接下来笔者的文章方向偏向于 Android & Java 面试相关知识点系统性的总结，欢迎关注。

`ThreadLocal`类是`java.lang`包下的一个类，用于线程内部的数据存储，通过它可以在指定的线程中存储数据，本文针对该类进行原理分析。

通过思维导图对其进行简单的总结：

![](https://user-gold-cdn.xitu.io/2019/3/31/169d448aae82e614?w=1732&h=988&f=png&s=215527)

## 一.ThreadLocal源码分析

`ThreadLocal`类最重要的几个方法如下：

* get():T         获取当前线程下存储的变量副本
* set(T):void     存储该线程下的某个变量副本
* remove():void   移除该线程下的某个变量副本

### 1.get()方法分析

`ThreadLocal`类比较简单，其最重要的就是`get()`和`set()`方法，顾名思义，起作用就是取值和设置值：

```java
// 获取当前线程中的变量副本
public T get() {
    // 获取当前线程
    Thread t = Thread.currentThread();
    // 获取线程中的ThreadLocalMap对象
    ThreadLocalMap map = getMap(t);
    if (map != null) {
        ThreadLocalMap.Entry e = map.getEntry(this);
        if (e != null) {
            @SuppressWarnings("unchecked")
            // 获取变量副本并返回
            T result = (T)e.value;
            return result;
        }
    }
    // 若没有该变量副本，返回setInitialValue()
    return setInitialValue();
}
```

这里先将`ThreadLocalMap`暂时理解为一个`Map`结构的容器，内部存储着该线程作用域下的的所有变量副本，我们从`ThreadLocal`类中取值的时候，实际上是从`ThreadLocalMap`中取值。

如果`Map`中没有该变量的副本，会从`setInitialValue()`中取值：

```Java
private T setInitialValue() {
   T value = initialValue();
   Thread t = Thread.currentThread();
   ThreadLocalMap map = getMap(t);
   if (map != null)
       map.set(this, value);
   else
       createMap(t, value);
   return value;
}
```

可以看到，`setInitialValue()`中也非常的简单，依然是从当前线程中获取到`ThreadLocalMap`，略微不同的是，`setInitialValue()`会对变量进行初始化，存入`ThreadLocalMap`中并返回。

这个初始化的方法的执行，需要开发者自己重写`initialValue()`方法，否则返回值依然为`null`。

```java
public class ThreadLocal<T> {
    // ...
    protected T initialValue() {
       return null;
    }
}    
```

### 2.set()方法分析

和`setInitialValue()`方法类似，`set()`方法也非常简单：
```java
public void set(T value) {
    Thread t = Thread.currentThread();
    ThreadLocalMap map = getMap(t);
    if (map != null)
        // map不为空，直接将ThreadLocal对象作为key
        // 变量本身的值为value，存入map
        map.set(this, value);
    else
        // 否则，创建ThreadLocalMap
        createMap(t, value);
}
```

可以看到，这个方法的作用就是将变量副本作为`value`存入`Map`，需要注意的是，`key`并非是我们下意识认为的`Thread`对象，而是`ThreadLocal`本身（`Thread`和`Value`本身是一对一的，我们更容易将其映射为`key-value`的关系）。

### 3.remove()方法分析

```Java
public void remove() {
   ThreadLocalMap m = getMap(Thread.currentThread());
   if (m != null)
       m.remove(this);
}
```

对于变量副本的移除，也是通过`map`进行处理的，和`set()`和`get()`相同，`Entry`的键值对中，`ThreadLocal`本身作为`key`，对变量副本进行检索。

### 4.小结

可以看出，`ThreadLocal`本身内部的逻辑都是围绕着`ThreadLocalMap`在运作，其本身更像是一个空壳，仅作为`API`供开发者调用，内部逻辑都委托给了`ThreadLocalMap`。

接下来我们来探究一下`ThreadLocalMap`和`Thread`以及`ThreadLocal`之间的关系。

## 二、ThreadLocalMap分析

`ThreadLocalMap`内部代码和算法相对复杂，个人亦是一知半解，因此就不逐行代码进行分析，仅系统性进行概述。

首先来看一下`ThreadLocalMap`的定义：

```Java
public class ThreadLocal<T> {

    // ThreadLocalMap是ThreadLocal的内部类
    static class ThreadLocalMap {

      // Entry类，内部key对应的是ThreadLocal的弱引用
      static class Entry extends WeakReference<ThreadLocal<?>> {
          // 变量的副本，强引用
          Object value;

          Entry(ThreadLocal<?> k, Object v) {
              super(k);
              value = v;
          }
      }
    }
}
```

`ThreadLocal`中的嵌套内部类`ThreadLocalMap`本质上是一个`map`，依然是`key-value`的形式，其中有一个内部类`Entry`，其中`key`可以看做是`ThreadLocal`实例的弱引用。

和最初的设想不同的是，`ThreadLocalMap`中`key`并非是线程的实例`Thread`，而是`ThreadLocal`，那么`ThreadLocalMap`是如何保证同一个`Thread`中，`ThreadLocal`的指定变量唯一呢？

```Java
// 1.ThreadLocal的set()方法
public void set(T value) {
    Thread t = Thread.currentThread();
    ThreadLocalMap map = getMap(t);
    // ...
}

// 2.getMap()实际上是从Thread中获取threadLocals成员
ThreadLocalMap getMap(Thread t) {
    return t.threadLocals;
}

public class Thread implements Runnable {
    // 3.每个Thread实例都持有一个ThreadLocalMap的属性
    ThreadLocal.ThreadLocalMap threadLocals = null;
}
```

`Thread`本身持有`ThreadLocal.ThreadLocalMap`的属性，每个线程在向`ThreadLocal`里`setValue`的时候，其实都是向自己的`ThreadLocalMap`成员中加入数据；`get()`同理。

## 三、内存泄漏的风险？

在上一小节中，我们看到`ThreadLocalMap`中的`Entry`中，其`ThreadLocal`作为`key`，是作为弱引用进行存储的。

当`ThreadLocal`不再被作为强引用持有时，会被GC回收，这时`ThreadLocalMap`对应的`ThreadLocal`就变成了`null`。而根据文档所叙述的，当`key == null`时，这时就可以默认该键不再被引用，该`Entry`就可以被直接清除，该清除行为会在`Entry`本身的`set()/get()/remove()`中被调用，这样就能 **一定情况下避免内存泄漏**。

这时就有一个问题出现了，作为`key`的`ThreadLocal`变成了`null`，那么作为`value`的变量可是强引用呀，这不就导致内存泄漏了吗？

其实一般情况下也不会，因为即使再不济，线程在执行结束时，自然也会消除其对`value`的引用，使得`Value`能够被GC回收。

当然，在某种情况下（比如使用了 **线程池**），线程再次被使用，`Value`这时依然可以被获取到，自然也就发生了内存泄漏，因此此时，我们还是需要通过手动将`value`的值设置为`null`（即调用`ThreadLocal.remove()`方法）以规避内存泄漏的风险。

## 参考&感谢

* 《Android开发艺术探索》
* [深入理解ThreadLocal的"内存溢出"](https://mahl1990.iteye.com/blog/2347932)
* [关于ThreadLocal内存泄露的备忘](https://www.jianshu.com/p/250798f9ff76)
* [ThreadLocal源码分析](https://www.jianshu.com/p/80866ca6c424)

---

## 关于我

Hello，我是[却把清梅嗅](https://github.com/qingmei2)，如果您觉得文章对您有价值，欢迎 ❤️，也欢迎关注我的[博客](https://www.jianshu.com/u/df76f81fe3ff)或者[Github](https://github.com/qingmei2)。

如果您觉得文章还差了那么点东西，也请通过**关注**督促我写出更好的文章——万一哪天我进步了呢？

* [我的Android学习体系](https://github.com/qingmei2/android-programming-profile)
* [关于文章纠错](https://github.com/qingmei2/Programming-life/blob/master/error_collection.md)
* [关于知识付费](https://github.com/qingmei2/Programming-life/blob/master/appreciation.md)
