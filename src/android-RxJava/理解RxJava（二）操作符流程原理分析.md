## 概述

在我的[上一篇文章《理解RxJava（一）基本流程源码分析》](http://blog.csdn.net/mq2553299/article/details/78670164)
中，通过Observable.create().subscribe()的原理进行了简单的分析。

今天尝试对多个操作符的链式调用进行分析，示例代码：

```Java
    @Test
    public void test() throws Exception {
        Observable.create((ObservableOnSubscribe<Integer>) e -> {
            e.onNext(1);
            e.onNext(2);
            e.onComplete();
        }).map(new Function<Integer, Integer>() {
            @Override
            public Integer apply(Integer i) throws Exception {
                return i + 10;
            }
        })
          .doOnNext(new Consumer<Integer>() {
              @Override
              public void accept(Integer i) throws Exception {
                      System.out.println("doOnNext : i= " + i);
              }
          })
          .subscribe(i -> System.out.println("onNext : i= " + i));
    }
```
输出结果：

> doOnNext : i= 11
onNext : i= 11
doOnNext : i= 12
onNext : i= 12

## 回溯

先回顾 [ 上文 ](http://blog.csdn.net/mq2553299/article/details/78670164)的内容：

* 1.Observable.create()，实例化ObservableCreate和传入最上游数据源ObservableOnSubscribe
* 2.Observable.subscribe(),实例化ObservableEmitter,负责发射数据
* 3.执行Observer.onSubscribe()回调
* 4.执行ObservableOnSubscribe.subscribe()方法,依次发射数据

## Map操作符

回到本文中案例代码，我们以使用频率最高的map操作符先进行探讨。

先看一下map()方法的代码内部：

```Java
    public final <R> Observable<R> map(Function<? super T, ? extends R> mapper) {
        //...省略非核心代码，下同
        return RxJavaPlugins.onAssembly(new ObservableMap<T, R>(this, mapper));
    }
```
仍旧先不讨论RxJava内部使用的钩子机制（hook），默认map内部方法执行的是，创建一个ObservableMap的对象：

```Java
//1.ObservableMap类继承了AbstractObservableWithUpstream类
public final class ObservableMap<T, U> extends AbstractObservableWithUpstream<T, U> {
    final Function<? super T, ? extends U> function;
  
    //2.传入Function转换函数
    public ObservableMap(ObservableSource<T> source, Function<? super T, ? extends U> function) {
        super(source);
        this.function = function;
    }
    
  //3.实际订阅时执行的方法
    @Override
    public void subscribeActual(Observer<? super U> t) {
        source.subscribe(new MapObserver<T, U>(t, function));
    }
}
```
这个类有三点值得注意，我已经注释在源码中。

### AbstractObservableWithUpstream

首先，来看继承的AbstractObservableWithUpstream类：

```Java
//实际上这个类的本质还是继承的Observable
abstract class AbstractObservableWithUpstream<T, U> extends Observable<U> implements HasUpstreamObservableSource<T> {
    
    //这个成员变量存储的是上游的Observable
    protected final ObservableSource<T> source;

    AbstractObservableWithUpstream(ObservableSource<T> source) {
        this.source = source;
    }

    @Override
    public final ObservableSource<T> source() {
        return source;
    }

}
```
和ObservableCreate不同的是，ObservableMap继承了AbstractObservableWithUpstream这个Observable的装饰器，这个类的作用人如其名，就是用来“存储上游Observable”的Observable,这意味着我们目前可以把Observable和其装饰器分为两种：

* 第一种，最上游数据源 型Observable，类似ObservableCreate，ObservableJust等等。
* 第二种，类似ObservableMap，ObservableDoOnEach(下文马上讲到)，这些类实际上继承了Observable的装饰器AbstractObservableWithUpstream，而不是直接继承Observable。其作用就是在初始化的同时，将上游的Observable数据源作为成员变量存储起来。

总结一句话，第一种是起点Observable，第二种则是过程Observable。

### Function函数

Function函数是Java8里面的一个接口：

```
public interface Function<T, R> {
    R apply(@NonNull T t) throws Exception;
}

//实际上就是本文案例代码中的：
new Function<Integer, Integer>() {
      @Override
      public Integer apply(Integer i) throws Exception {
            return i + 10;
      }
}
```
其作用就是把上游的数据进行变换，因此我们看到，在源码中，这个Function函数也被当做成员进行了存储，在订阅的时候通过ObservableMap.subscribeActual()作为参数传入。

### subscribeActual()

由上文我们知道，当我们执行Observable.subscrbe(Observer o)的时候，实际上执行的就是对应的subscribeActual()方法：

```Java
    public final void subscribe(Observer<? super T> observer) {
        //...
        try {
            //...
            subscribeActual(observer);
        } catch (NullPointerException e) { 
            //...
        } catch (Throwable e) {
            //...
        }
    }
```
当ObservableMap执行subscribeActual()时，我们接收到一个下游的Observer参数，并将其和内部存储的function函数一起作为参数实例化了一个MapObserver对象，并交给上游数据源Observable进行订阅。

#### 数据流的上游和下游

关于这个所谓“下游的Observer”，一定要理解，所谓上游下游，指的就是距离数据源的远近，这个是相对的，在本文案例中，很明显，Observable.Create相对其他操作就是上游，Observable.Map相对Observable.Create是下游，但是相对Observable.doOnNext和Observable.subscribe就是上游；当然，Observable.subscribe是订阅处，相对其它操作都是下游。

### MapObserver

看一下MapObserver对象,这个类有很多有意思的地方，初学源码不多深入，先大概看一下onNext()的执行流程：

```Java
static final class MapObserver<T, U> extends BasicFuseableObserver<T, U> {
        final Function<? super T, ? extends U> mapper;

        MapObserver(Observer<? super U> actual, Function<? super T, ? extends U> mapper) {
            super(actual);
            this.mapper = mapper;
        }

        @Override
        public void onNext(T t) {
            //....
            U v;

            try {
                // 1. function数据变换
                v = ObjectHelper.requireNonNull(mapper.apply(t), "The mapper function returned a null value.");
            } catch (Throwable ex) {
                fail(ex);
                return;
            }
            // 2.执行Observer.onNext()
            actual.onNext(v);
        }
        //...
    }
```
很好理解，实际上MapObserver对象在执行onNext的时候，先执行funtion函数进行数据变换，将转换后的数据，传给下游传入的Observer，执行下游Observer的onNext方法。

> 值得一提的是，我们可以看到，在数据进行了转换之后，会将转换后的数据进行一次ObjectHelper.requireNonNull()非空校验，如果转换后的数据为空，则会抛出错误，这一点我们一定要注意，数据流的传递和转换过程中，一定要避免null值的情况。

关于Map操作符的原理分析先到此为止，我们接下来看一下doOnNext操作符。

## doOnNext操作符

直接看内部代码：

```Java
    public final Observable<T> doOnNext(Consumer<? super T> onNext) {
        //实际上执行的是doOnEach方法
        return doOnEach(onNext, Functions.emptyConsumer(), Functions.EMPTY_ACTION, Functions.EMPTY_ACTION);
    }
```

我们看一下doOnEach()方法：
```Java
    private Observable<T> doOnEach(Consumer<? super T> onNext, Consumer<? super Throwable> onError, Action onComplete, Action onAfterTerminate) {
        ObjectHelper.requireNonNull(onNext, "onNext is null");
        ObjectHelper.requireNonNull(onError, "onError is null");
        ObjectHelper.requireNonNull(onComplete, "onComplete is null");
        ObjectHelper.requireNonNull(onAfterTerminate, "onAfterTerminate is null");
        return RxJavaPlugins.onAssembly(new ObservableDoOnEach<T>(this, onNext, onError, onComplete, onAfterTerminate));
    }
```
很简单，在进行简单的非空校验后，doOnNext()实际上是直接生成一个ObservableDoOnEach对象并返回。

这个onNext参数就是我们案例中的：

```
new Consumer<Integer>() {
      @Override
      public void accept(Integer i) throws Exception {
              System.out.println("doOnNext : i= " + i);
       }
  }
```

### ObservableDoOnEach

有了Map操作符的经验，我们轻车熟路：

```Java
//1.同样继承AbstractObservableWithUpstream
public final class ObservableDoOnEach<T> extends AbstractObservableWithUpstream<T, T> {
    final Consumer<? super T> onNext;
    final Consumer<? super Throwable> onError;
    final Action onComplete;
    final Action onAfterTerminate;
  
    //2.构造器，存储对应函数，以及上游的Observable
    public ObservableDoOnEach(ObservableSource<T> source, Consumer<? super T> onNext,
                              Consumer<? super Throwable> onError,
                              Action onComplete,
                              Action onAfterTerminate) {
        super(source);
        this.onNext = onNext;  //就是System.out.println("doOnNext : i= " + i);
        //这三个都是以默认参数传进来的
        this.onError = onError;
        this.onComplete = onComplete;
        this.onAfterTerminate = onAfterTerminate;
    }
  
    //实际订阅执行的方法
    @Override
    public void subscribeActual(Observer<? super T> t) {
        source.subscribe(new DoOnEachObserver<T>(t, onNext, onError, onComplete, onAfterTerminate));
    }
}
```

在案例代码中，doOnNext()在未subscribe订阅之前，将上游的数据源ObservableMap作为source成员变量存储，同时存储ObservableMap数据到来时，将要执行的函数onNext()

千万注意，这个onNext()不是我们subscribe时传入的onNext()，而是doOnNext()传入的：

> System.out.println("doOnNext : i= " + i);

### DoOnEachObserver

我们看下subscribeActual方法内部生成的这个DoOnEachObserver是何方神圣：

```
static final class DoOnEachObserver<T> implements Observer<T>, Disposable {
        final Observer<? super T> actual;
        final Consumer<? super T> onNext;
        final Consumer<? super Throwable> onError;
        final Action onComplete;
        final Action onAfterTerminate;

        Disposable s;

        boolean done;
        
        //1.构造器，实际上就是依赖注入
        DoOnEachObserver(
                Observer<? super T> actual,
                Consumer<? super T> onNext,
                Consumer<? super Throwable> onError,
                Action onComplete,
                Action onAfterTerminate) {
            this.actual = actual;
            this.onNext = onNext;
            this.onError = onError;
            this.onComplete = onComplete;
            this.onAfterTerminate = onAfterTerminate;
        }
        
        //2.执行onNext
        @Override
        public void onNext(T t) {
            if (done) {
                return;
            }
            try {
         //3.执行System.out.println("doOnNext : i= " + i);
                onNext.accept(t);
            } catch (Throwable e) {
                Exceptions.throwIfFatal(e);
                s.dispose();
                onError(e);
                return;
            }
         //4.执行下游Observer的onNext
            actual.onNext(t);
        }
        //......先忽略大部分代码，我们暂时只关注onNext
    }
```

先忽略线程调度和其他参数及相关方法，我们只关注onNext，实际上就是先执行doOnNext()：

> System.out.println("doOnNext : i= " + i);

然后执行下游Observer的onNext，也就是：

>  System.out.println("onNext : i= " + i);

这也正好契合了我们上文的输出结果：

>doOnNext : i= 11
onNext : i= 11
doOnNext : i= 12
onNext : i= 12

## 流程整理

其实到这里，还是有些乱，上游下游可能还是不能很好的看清。

先将代码再发一次：

```
    @Test
    public void test() throws Exception {
        Observable.create((ObservableOnSubscribe<Integer>) e -> {
            e.onNext(1);
            e.onNext(2);
            e.onComplete();
        }).map(new Function<Integer, Integer>() {
            @Override
            public Integer apply(Integer i) throws Exception {
                return i + 10;
            }
        })
          .doOnNext(new Consumer<Integer>() {
              @Override
              public void accept(Integer i) throws Exception {
                      System.out.println("doOnNext : i= " + i);
              }
          })
          .subscribe(i -> System.out.println("onNext : i= " + i));
    }
```

我们将创建和订阅分两步来讲：

### 创建

在未执行subscribe订阅之前，执行的流程应该是这样的：

本文借鉴图片来源，下同：[RxJava2 源码解析——流程 @Robin_Lrange](http://www.jianshu.com/p/e5be2fa8701c)

![](http://upload-images.jianshu.io/upload_images/5305830-63c58ab311ad3c58.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

图片是借鉴的，我们选择性忽略线程调度的两步，其他基本都是一致的。

### 订阅

订阅时，执行的流程变成了：

![](http://upload-images.jianshu.io/upload_images/5305830-b40e8d9d475fb33a.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

总结来说，当执行subscribe时，Observable中存储的上游的Observable 会被下游的Observer订阅：

```Java
    //以ObservableDoOnEach为例
    @Override
    public void subscribeActual(Observer<? super T> t) {
        source.subscribe(new DoOnEachObserver<T>(t, onNext, onError, onComplete, onAfterTerminate));
    }
```
可以看到，source是上游的ObservableMap,它被下游的DoOnEachObserver订阅。

我们不要忽视这个source实际上也是Observable的装饰器（ObserverMap），它也会执行其内部的subscribeActual方法：

```
    //ObservableMap的subscribeActual
    @Override
    public void subscribeActual(Observer<? super U> t) {
        source.subscribe(new MapObserver<T, U>(t, function));
    }
```

是不是理解过来了？实际上就是装饰器模式的体现,这就是所谓的：

> Observable中存储的上游的Observable 会被下游的Observer订阅。

即：

> 每一步都会生成对应的Observer对上一步生成并存储的Observable进行订阅。

这说明，在订阅时，实际上这个顺序是逆向的，从下游往上游进行订阅。

## 上游发射数据

当逆向订阅执行到最上游时，即ObservableCreate.subscribe()方法时，我们上文已经讲过了，会通过实例好的ObservableEmitter进行数据的发射，那么数据就会从上游到下游，一层层执行对应Observer的onNext方法：

![](http://upload-images.jianshu.io/upload_images/5305830-5de7b5e046e10b91.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

和订阅不同，数据的传递和变换则是正常，方向从上游往下游进行依次处理，最终执行我们subscribe中传递的Observer.onNext()。

## 总结

* 1.创建：订阅前，每一步都生成对应的Observable对象，中间的每一步都将上游的Observable存储；
* 2.订阅： 每一步都会生成对应的Observer对上一步生成并存储的Observable进行订阅。**订阅的执行顺序是由下到上的。**
* 3.执行：先执行每一步传入的函数操作，然后将操作后的数据交给下游的Observer继续处理。 **数据的传递和处理顺序是由上到下的。**

接下来我会尝试对RxJava最核心的线程调度的原理进行解析。

## 参考文章

1.[RxJava2 源码解析——流程 @Robin_Lrange](http://www.jianshu.com/p/e5be2fa8701c)

感谢[@Robin_Lrange](http://www.jianshu.com/u/d776817c7fc7)的精彩文章和图片！

2.[RxJava2 源码解析（二）@张旭童](http://www.jianshu.com/p/6ef45f8ee79d)

