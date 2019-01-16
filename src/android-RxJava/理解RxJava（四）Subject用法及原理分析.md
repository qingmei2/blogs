## 概述

本文将尽可能将RxJava中的Subject相关类的用法做一个阐述，并对其原理进行简单的解析。

说到Subject，很多人可能都不是很熟悉它，因为相对于RxJava的Observable、Schedulers、Subscribes等关键字来讲，它抛头露面的场合似乎很少。

事实上，Subject作用是很大的，借用官方的解释，Subject在同一时间内，既可以作为Observable，也可以作为Observer:

在RxJava2.x中，官方一共为我们提供了以下几种Subject：

* ReplaySubject （释放接收到的所有数据）
* **BehaviorSubject** （释放订阅前最后一个数据和订阅后接收到的所有数据）
* **PublishSubject** （释放订阅后接收到的数据）
* AsyncSubject  （仅释放接收到的最后一个数据）
* *SerializedSubject*（串行Subject）
* *UnicastSubject* (仅支持订阅一次的Subject)
* *TestSubject*（已废弃，在2.x中被TestScheduler和TestObserver替代）

下面依次对以上的Subject进行讲解，本文将重点讲解**BehaviorSubject**和**PublishSubject**。前者是[RxLifecycle](https://github.com/trello/RxLifecycle)中核心类所使用到，后者则是RxBus实现事件总线的核心类。

在开始正文之前，我们需要搞清楚一个问题：

## Subject是什么？

> Subject在ReactiveX是作为observer和observerable的一个bridge或者proxy。因为它是一个观察者，所以它可以订阅一个或多个可观察对象，同时因为他是一个可观测对象，所以它可以传递和释放它观测到的数据对象，并且能释放新的对象。

上文已经说的很清楚了，它既可以是数据源**observerable**，也可以是数据的订阅者**Observer**。

我们点开源码：

```Java
public abstract class Subject<T> extends Observable<T> implements Observer<T> {
    ...
}
```

可以看到，Subject实际上还是Observable，只不过它继承了Observer接口，可以通过onNext、onComplete、onError方法发射和终止发射数据。

我们从不同种类的Subject展开来讲：

## 一、ReplaySubject

### 介绍

> 该Subject会接收数据，当被订阅时，将所有接收到的数据全部发送给订阅者。

这意味着，不管何时订阅这个Subject，这个Subject会把它接收到的数据都发送出去：

```Java
  ReplaySubject<Object> subject = new ReplaySubject<>();
  subject.onNext("one");
  subject.onNext("two");
  subject.onNext("three");
  subject.onComplete();

  // both of the following will get the onNext/onComplete calls from above
  subject.subscribe(observer1);
  subject.subscribe(observer2);
```

 一图顶千言,下方的箭头代表两次不同时间的订阅：

![ReplaySubject](http://upload-images.jianshu.io/upload_images/7293029-670ffea893daf9d2?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

很显然，无论是在接收到数据前还是数据后订阅，ReplaySubject都会发射所有数据给订阅者。

### 原理

> 事实上，每个Subject的实现都不可避免的涉及到线程安全的问题，而RxJava则是依靠在内部使用原子操作类（AtomicXXX系列）保证线程安全，本文不深入讨论原子操作类相关知识，有兴趣的朋友可以参考：
> [深入分析Java中的原子操作 @码农一枚](http://www.jianshu.com/p/9ff426a784ad)

ReplaySubject类时Subject相关类中代码最多的一个类，但这并不意味着这个类是最难理解的，相反，它的原理很直白：通过一个List动态存储所有接收到的数据，当被订阅时，将所有的数据都发送给订阅者。

是不是觉得很熟悉？是的，其根本就是一个动态的链表，甚至其创建时的基础容量也是16，并且随着数据的不断增加，每次递增50%，如果想要节省开支，也可以自己定义初始容量和递增规则。

千余行的代码并非上述总结的那么简单，不过作为了解和使用已经足够，事实上，这其中绝大部分都是算法和数据结构的处理，笔者研究源码时，也并没有去深入分析源码，只是浅尝辄止。

## 二、BehaviorSubject

> 当Observer订阅了一个BehaviorSubject，它一开始就会释放Observable最近释放的一个数据对象，当还没有任何数据释放时，它则是一个默认值。接下来就会释放Observable释放的所有数据。如果Observable因异常终止，BehaviorSubject将不会向后续的Observer释放数据，但是会向Observer传递一个异常通知。

简单来说，就是释放**订阅前最后一个数据**和**订阅后**接收到的**所有数据**:

```
  // observer will receive all 4 events (including "default").
  BehaviorSubject<Object> subject = BehaviorSubject.createDefault("default");
  subject.subscribe(observer);
  subject.onNext("one");
  subject.onNext("two");
  subject.onNext("three");

  // observer will receive the "one", "two" and "three" events, but not "zero"
  BehaviorSubject<Object> subject = BehaviorSubject.create();
  subject.onNext("zero");
  subject.onNext("one");
  subject.subscribe(observer);
  subject.onNext("two");
  subject.onNext("three");

  // observer will receive only onComplete
  BehaviorSubject<Object> subject = BehaviorSubject.create();
  subject.onNext("zero");
  subject.onNext("one");
  subject.onComplete();
  subject.subscribe(observer);

  // observer will receive only onError
  BehaviorSubject<Object> subject = BehaviorSubject.create();
  subject.onNext("zero");
  subject.onNext("one");
  subject.onError(new RuntimeException("error"));
  subject.subscribe(observer);
```

所有case都通过上述代码进行了分析，我们看下图：

![BehaviorSubject](http://upload-images.jianshu.io/upload_images/7293029-45046b825f329b33?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

### 原理

我们简单看一下BehaviorSubject内部的几个成员：

```
public final class BehaviorSubject<T> extends Subject<T> {

    private static final Object[] EMPTY_ARRAY = new Object[0];
    //原子操作类，当前接收到的最后一个数据
    final AtomicReference<Object> value;
  
    //原子操作类，BehaviorDisposable内部存储了所有接受到的数据
    final AtomicReference<BehaviorDisposable<T>[]> subscribers;
    
    //标记，意味着一个空的BehaviorDisposable
    static final BehaviorDisposable[] EMPTY = new BehaviorDisposable[0];
    
    //标记，意味着已经达到了TERMINATED，终止数据的发射
    static final BehaviorDisposable[] TERMINATED = new BehaviorDisposable[0];
...
}
```

我们再来看一下BehaviorDisposable：

```
    static final class BehaviorDisposable<T> implements Disposable, NonThrowingPredicate<Object> {
      ...
      //实际上内部有一个List，存储所有接收到的数据
      AppendOnlyLinkedArrayList<Object> queue;
      ...
}
```

其原理是：

1.当BehaviorSubject.create()实例化时，默认将EMPTY赋值给subscribers，这意味着此时Subject中没有数据，：

```
    BehaviorSubject() {
        //...省略其他代码
        //这意味着Subject中没有数据
        this.subscribers = new AtomicReference<BehaviorDisposable<T>[]>(EMPTY);  
        //当前没有最终的数据
        this.value = new AtomicReference<Object>();
    }
```

2.当通过onNext发射数据时，存储数据：
```
    @Override
    public void onNext(T t) {
        //省略判断代码
        Object o = NotificationLite.next(t);
        setCurrent(o);    //内部就是给成员value赋值，记录最新接收到的数据
        for (BehaviorDisposable<T> bs : subscribers.get()) {
            bs.emitNext(o, index);  //其实就是EMPTY.emitNext()
        }
    }
```
3.当订阅时，执行subscribeActual方法：

```
    @Override
    protected void subscribeActual(Observer<? super T> observer) {
        BehaviorDisposable<T> bs = new BehaviorDisposable<T>(observer, this);
        observer.onSubscribe(bs);
        //1.执行add(bs)方法
        if (add(bs)) {
            if (bs.cancelled) {
                remove(bs);//2.如果已经取消发射，则不中断数据的传递
            } else {
                bs.emitFirst();//3.执行bs.emitFirst方法
            }
        } else {
            //....
        }
    }
```
看一下add方法：

```
    boolean add(BehaviorDisposable<T> rs) {
        for (;;) {
            BehaviorDisposable<T>[] a = subscribers.get();
            if (a == TERMINATED) {
                return false;
            }
            int len = a.length;
            BehaviorDisposable<T>[] b = new BehaviorDisposable[len + 1];
            System.arraycopy(a, 0, b, 0, len);
            b[len] = rs;
            //原子数据的更新操作
            if (subscribers.compareAndSet(a, b)) {
                return true;
            }
        }
    }
```
不熟悉原子操作类的同学，我简单介绍一下上述代码的思想：
首先获取subscribers的值（应该获得一个BehaviorDisposable数组，其中有且仅有一个元素EMPTY）,然后将该数组扩容+1，将新创建的BehaviorDisposable对象放入数组index=1的位置上，并将该数组赋值给subscriber。

> 可以看到，RxJava中的源码并没有使用Java Util包中的Collections相关工具类进行数据的操作，而是直接使用System.arraycopy(a, 0, b, 0, len)这个Native底层方法进行数组的处理，减少了额外的开支。

至于BehaviorDisposable的emitFirst方法，我们不难想象，应该是处理数据的发射相关逻辑，实际上它会发射BehaviorSubject的value，这也就是为什么当订阅后，observer会**先接收到订阅前的最后一个数据**的原因。

这之后，每当onNext（）从上游接收到一个数据，都会subscribers数组里对每一个BehaviorDisposable中的observer向下发射数据。

### BehaviorSubject小结
其原理就是通过subscribers这个核心的成员，它是一个不断变化的数组。在创建时，其内部只是一个EMPTY（BehaviorDisposable）对象，每次被订阅，都会在既有的数组上新加一个BehaviorDisposable对象，这个对象中包含了一个List，存储之后会收到的数据。

同时，BehaviorSubject还有一个value的成员，该成员会随着数据的不断接收而进行更新，它总是记录着当前最后一个接收到的数据，当被subscribe时，会执行emitFirst()方法，发射当前记录的数据，也就是订阅前接收到的最后一个数据。

## 三、PublishSubject
> PublishSubject仅会向Observer释放在订阅之后Observable释放的数据。

参考前两个Subject的子类，这个类很好理解，先上代码：

```
  PublishSubject<Object> subject = PublishSubject.create();
  // observer1 will receive all onNext and onComplete events
  subject.subscribe(observer1);
  subject.onNext("one");
  subject.onNext("two");

  // observer2 will only receive "three" and onComplete
  subject.subscribe(observer2);
  subject.onNext("three");
  subject.onComplete();
```
用图说话：

![PublishSubject](http://upload-images.jianshu.io/upload_images/7293029-e01605bbda88ee49?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

### 原理
和BehaviorSubject原理思想如出一辙， 而PublishSubject更简单，其内部的PublishDisposable的原子操作类连AtomicReference都不是，而是AtomicBoolean：

```
public final class PublishSubject<T> extends Subject<T> {
    static final PublishDisposable[] TERMINATED = new PublishDisposable[0];
    static final PublishDisposable[] EMPTY = new PublishDisposable[0];

    final AtomicReference<PublishDisposable<T>[]> subscribers;
}
```

```
static final class PublishDisposable<T> extends AtomicBoolean implements Disposable {
}
```
> 其原理就是通过subscribers这个核心的成员，它是一个不断变化的数组。在创建时，其内部只是一个EMPTY（PublishDisposable）对象，每次被订阅，都会在既有的数组上新加一个PublishDisposable对象，这个对象中包含了一个List，存储之后会收到的数据。

其实往上翻，你会发现，这段总结是我复制BehaviorSubject小结的第一段......不是（其实就是）偷懒，而是两者本身几乎没什么区别，PublishDisposable更简单。

### 尝试放弃RxBus !!!

为什么要着重说PublishSubject这个类，原因很简单，因为在目前流行的事件总线处理方式RxBus中，其原理就是使用PublishSubject进行数据的分发。

RxBus的代码基本如下：

```
public class RxBus {

    private static RxBus rxBus;
    private final Subject<Object, Object> _bus = new SerializedSubject<>(PublishSubject.create());

    private RxBus() {
    }

    public static RxBus getInstance() {
        if (rxBus == null) {
            synchronized (RxBus.class) {
                if (rxBus == null) {
                    rxBus = new RxBus();
                }
            }
        }
        return rxBus;
    }

    public void send(Object o) {
        _bus.onNext(o);
    }

    public Observable<Object> toObserverable() {
        return _bus;
    }
}
```
每个人的RxBus类略有不同，可能有的RxBus类会添加类似ofType()相关的filter操作符对数据进行筛选，但其原理基本一模一样，都是通过PublishSubject进行数据的接收和分发，这样的使用会有什么后果，请参考下文：

[放弃RxBus，拥抱RxJava（一）：为什么避免使用EventBus/RxBus](http://www.jianshu.com/p/61631134498e)

文中作者已经很形象举出了RxBus的种种缺陷，以及对于事件的传递应该如何处理。

在分析完PublishSubject的原理后，我们更应该清楚，RxBus类似一个全局的Observable，不断地接受新的数据，以及接受订阅，如果使用不当，会导致代码的混乱，我们也应该考虑，在接受的数据和订阅越来越多，PublishSubject的单例对象会占用越来越多的内存，如果没有处理好生命周期相关，则会导致严重的内存泄漏！

因此，已经分析完RxBus（实际上也就是PublishSubject ）后，我们应当对其使用的方式有一个全新的认知，使用，亦或放弃RxBus。

## 四、AsyncSubject

> AsyncSubject仅释放Observable释放的最后一个数据，并且仅在Observable完成之后。然而如果当Observable因为异常而终止，AsyncSubject将不会释放任何数据，但是会向Observer传递一个异常通知。

随着可以接收到数据的范围越来越小，似乎也越来越好理解了，直接看图：

![AsyncSubject](http://upload-images.jianshu.io/upload_images/7293029-e487a3bf939bf13e?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

## 原理

其原理就更简单了，这下原子操作了连AtomicBoolean都不是了，直接就是AtomicInteger：

```
public final class AsyncSubject<T> extends Subject<T> {

    static final AsyncDisposable[] EMPTY = new AsyncDisposable[0];

    static final AsyncDisposable[] TERMINATED = new AsyncDisposable[0];

    final AtomicReference<AsyncDisposable<T>[]> subscribers;
    //记录最新的数据
    T value;
}
```

```
//继承DeferredScalarDisposable
static final class AsyncDisposable<T> extends DeferredScalarDisposable<T> {}

//继承BasicIntQueueDisposable
public class DeferredScalarDisposable<T> extends BasicIntQueueDisposable<T> {}

//继承AtomicInteger
public abstract class BasicIntQueueDisposable<T>
extends AtomicInteger
implements QueueDisposable<T> {}
```
每当执行onComplete（）的时候，都会执行complete（Object o）方法,先发送最后一个数据，然后执行onComplete()：
```
    public final void complete(T value) {
        //......
        Observer<? super T> a = actual;
        //先发送最后一个数据
        a.onNext(value);
        if (get() != DISPOSED) {
            //执行onComplete()
            a.onComplete();
        }
    }
```

## 五、简单介绍UnicastSubject和SerializedSubject

### UnicastSubject

> 仅支持订阅一次的Subject,如果多个订阅者试图订阅这个Subject，若该subject未terminate，将会受到IllegalStateException ，若已经terminate，那么只会执行onError或者onComplete方法。

### SerializedSubject

回到刚刚的RxBus类中，我们发现了SerializedSubject的身影，实际上，它的作用就是：

> 将Subject串行化的方法，所有其他的Observable和Subject方法都是线程安全的。

我们也可以通过Subject.toSerialized()方法将Subject对象串行化保证其线程安全：

```
public abstract class Subject<T> extends Observable<T> implements Observer<T> {
    //......
    public final Subject<T> toSerialized() {
        if (this instanceof SerializedSubject) {
            return this;
        }
        return new SerializedSubject<T>(this);
    }
}
```
我们直接打开这个类，会发现，其实该类中，大部分代码都会通过**synchronized**加锁，这意味着会有额外的性能消耗。

这也更进一步说明了，使用RxBus，会额外增加更多的支出（因为RxBus中PublisSuject本身的单例对象就是调用了toSerialized()方法保证线程安全）。


