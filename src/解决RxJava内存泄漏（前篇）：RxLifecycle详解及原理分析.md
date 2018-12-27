## 前言

随着RxJava及RxAndroid的逐渐推广，使用者越来越多，但是有一个问题，RxJava的使用不当极有可能会导致内存泄漏。

> 比如，使用RxJava发布一个订阅后，当Activity被finish，此时订阅逻辑还未完成，如果没有及时取消订阅，就会导致Activity无法被回收，从而引发内存泄漏。

目前网上对RxJava的内存泄漏有几种方案：

1、通过封装，手动为RxJava的每一次订阅进行控制，在指定的时机进行取消订阅；
2、使用 [Daniel Lew](https://github.com/dlew) 的 [RxLifecycle](https://github.com/trello/RxLifecycle) ，通过监听Activity、Fragment的生命周期，来自动断开subscription以防止内存泄漏。

笔者上述两种方式都使用过，**RxLifecycle**显然对于第一种方式，更简单直接，并且能够在Activity/Fragment容器的**指定生命周期**取消订阅，实在是好用。


## 依赖并使用RxLifecycle

### 1、添加依赖

首先在build.gradle文件中添加依赖：
```groovy
compile 'com.trello.rxlifecycle2:rxlifecycle:2.2.1'
compile 'com.trello.rxlifecycle2:rxlifecycle-android:2.2.1'
compile 'com.trello.rxlifecycle2:rxlifecycle-components:2.2.1'
```

### 2、配置Activity/Fragment容器

Activity/Fragment需继承RxAppCompatActivity/RxFragment，目前支持的有如下：

 ![RxLifecycle支持的Component](http://upload-images.jianshu.io/upload_images/7293029-ce0d3865a740ac90.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

代码如下：

```Java
//只需要继承即可
public class MainActivity extends RxAppCompatActivity {
      ...
      ...
}
```

### 3、使用compose操作符绑定容器生命周期

有两种方式：

#### 3.1 使用bindToLifecycle()
以Activity为例，在Activity中使用bindToLifecycle()方法，完成Observable发布的事件和当前的组件绑定，实现生命周期同步。从而实现当前组件生命周期结束时，自动取消对Observable订阅，代码如下：

```Java
public class MainActivity extends RxAppCompatActivity {
     
    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);

        // 当执行onDestory()时， 自动解除订阅
        Observable.interval(1, TimeUnit.SECONDS)
            .doOnDispose(new Action() {
                @Override
                public void run() throws Exception {
                    Log.i(TAG, "Unsubscribing subscription from onCreate()");
                }
            })
            .compose(this.<Long>bindToLifecycle())
            .subscribe(new Consumer<Long>() {
                @Override
                public void accept(Long num) throws Exception {
                    Log.i(TAG, "Started in onCreate(), running until onDestory(): " + num);
                }
            });
    }
}
```
#### 3.2 使用bindUntilEvent()

使用ActivityEvent类，其中的CREATE、START、 RESUME、PAUSE、STOP、 DESTROY分别对应生命周期内的方法。使用bindUntilEvent指定在哪个生命周期方法调用时取消订阅：

```Java
public class MainActivity extends RxAppCompatActivity {
    @Override
    protected void onResume() {
        super.onResume();
        Observable.interval(1, TimeUnit.SECONDS)
            .doOnDispose(new Action() {
                @Override
                public void run() throws Exception {
                    Log.i(TAG, "Unsubscribing subscription from onResume()");
                }
            })
              //bindUntilEvent()，内部传入指定生命周期参数
            .compose(this.<Long>bindUntilEvent(ActivityEvent.DESTROY))
            .subscribe(new Consumer<Long>() {
                @Override
                public void accept(Long num) throws Exception {
                    Log.i(TAG, "Started in onResume(), running until in onDestroy(): " + num);
                }
            });
    }
}
```

以上，仅仅需要三步：依赖、继承、compose操作符，即可完成在容器的指定生命周期内，RxJava的自动取消订阅。

## 原理分析

RxLifecycle的原理可以说非常简单。

我们直接看一下这行代码的内部原理：

>  Observable.compose(this.<Long>bindToLifecycle())

### 1、RxAppCompatActivity

```Java
public abstract class RxAppCompatActivity extends AppCompatActivity implements LifecycleProvider<ActivityEvent> {
    
    //1.实际上RxAppCompatActivity内部存储了一个BehaviorSubject
    private final BehaviorSubject<ActivityEvent> lifecycleSubject = BehaviorSubject.create();

    public final Observable<ActivityEvent> lifecycle() {
        return lifecycleSubject.hide();
    }

    public final <T> LifecycleTransformer<T> bindUntilEvent(@NonNull ActivityEvent event) {
        return RxLifecycle.bindUntilEvent(lifecycleSubject, event);
    }
    
    //2.实际上返回了一个LifecycleTransformer
    public final <T> LifecycleTransformer<T> bindToLifecycle() {
        return RxLifecycleAndroid.bindActivity(lifecycleSubject);
    }
    
    //3.Activity不同的生命周期，BehaviorSubject对象会发射对应的ActivityEvent
    @Override
    @CallSuper
    protected void onCreate(@Nullable Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        lifecycleSubject.onNext(ActivityEvent.CREATE);
    }

    @Override
    @CallSuper
    protected void onStart() {
        super.onStart();
        lifecycleSubject.onNext(ActivityEvent.START);
    }

    @Override
    @CallSuper
    protected void onResume() {
        super.onResume();
        lifecycleSubject.onNext(ActivityEvent.RESUME);
    }

    @Override
    @CallSuper
    protected void onPause() {
        lifecycleSubject.onNext(ActivityEvent.PAUSE);
        super.onPause();
    }

    @Override
    @CallSuper
    protected void onStop() {
        lifecycleSubject.onNext(ActivityEvent.STOP);
        super.onStop();
    }

    @Override
    @CallSuper
    protected void onDestroy() {
        lifecycleSubject.onNext(ActivityEvent.DESTROY);
        super.onDestroy();
    }
}

```

我们继承的RxAppCompatActivity，其内部实际上存储了一个**BehaviorSubject**，关于**BehaviorSubject**，实际上也还是一个Observable，不了解的朋友可以阅读笔者的前一篇文章,本文不再赘述：

[理解RxJava（四）Subject用法及原理分析](http://blog.csdn.net/mq2553299/article/details/78848773)

这个**BehaviorSubject**会在不同的生命周期发射不同的ActivityEvent，比如在onCreate()生命周期发射ActivityEvent.CREATE，在onStop()发射ActivityEvent.STOP。

在2中，我们可以看到，bindToLifecycle()方法实际返回了一个LifecycleTransformer，那么这个LifecycleTransformer是什么呢？

### 2、LifecycleTransformer

```
public final class LifecycleTransformer<T> implements ObservableTransformer<T, T>,
                                                      FlowableTransformer<T, T>,
                                                      SingleTransformer<T, T>,
                                                      MaybeTransformer<T, T>,
                                                      CompletableTransformer
{
    final Observable<?> observable;

    LifecycleTransformer(Observable<?> observable) {
        checkNotNull(observable, "observable == null");
        this.observable = observable;
    }

    @Override
    public ObservableSource<T> apply(Observable<T> upstream) {
        return upstream.takeUntil(observable);
    }
    //隐藏其余代码，这里只以Observable为例
}
```

熟悉Transformer的同学都应该能看懂，LifecycleTransformer实际是实现了不同响应式数据类（Observable、Flowable等）的Transformer接口。

以Observable为例，实际上就是通过传入一个Observable序列并存储为成员，然后作为参数给上游的数据源执行takeUntil方法。

#### takeUntil操作符

takeUntil操作符:当第二个Observable发射了一项数据或者终止时，丢弃原Observable发射的任何数据。

![takeUntil操作符](http://upload-images.jianshu.io/upload_images/7293029-8cbeb88eb8bfd7fa?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

#### 回顾

```
 Observable.interval(1, TimeUnit.SECONDS)
            .doOnDispose(new Action() {
                @Override
                public void run() throws Exception {
                    Log.i(TAG, "Unsubscribing subscription from onCreate()");
                }
            })
            .compose(this.<Long>bindToLifecycle())
            .subscribe(new Consumer<Long>() {
                @Override
                public void accept(Long num) throws Exception {
                    Log.i(TAG, "Started in onCreate(), running until onDestory(): " + num);
                }
            });
```
现在回头来看这段代码，就很好理解了，Activity.<Long>bindToLifecycle()实际上就是指定上游的数据源，当接收到某个Observable（就是LifecycleTransformer中那个神秘的成员变量）的某个事件时，该数据源自动解除订阅。

老师，原理已经搞清楚了，我还有最后一个问题：

### 神秘人是谁？

回到RxAppCompatActivity中来，我们来看bindToLifecycle()方法：

```
public abstract class RxAppCompatActivity extends AppCompatActivity implements LifecycleProvider<ActivityEvent> {

    private final BehaviorSubject<ActivityEvent> lifecycleSubject = BehaviorSubject.create();

    public final <T> LifecycleTransformer<T> bindToLifecycle() {
        //执行了这行代码，返回了LifecycleTransformer
        return RxLifecycleAndroid.bindActivity(lifecycleSubject);
    }
}
```

不难猜测，实际上，那个神秘人，就是我们RxAppCompatActivity 中的BehaviorSubject成员变量（它本身就是一个Observable）！

我们点进去看看：

```
    //1.执行了 bind(lifecycle, ACTIVITY_LIFECYCLE);
    public static <T> LifecycleTransformer<T> bindActivity(@NonNull final Observable<ActivityEvent> lifecycle) {
        return bind(lifecycle, ACTIVITY_LIFECYCLE);
    }
    
    //2.执行了bind(takeUntilCorrespondingEvent(lifecycle.share(), correspondingEvents))
    public static <T, R> LifecycleTransformer<T> bind(@Nonnull Observable<R> lifecycle,
                                                      @Nonnull final Function<R, R> correspondingEvents) {
        return bind(takeUntilCorrespondingEvent(lifecycle.share(), correspondingEvents));
    }

```

```
    //3.最终抵达这里，这个方法执行了什么呢？
    private static <R> Observable<Boolean> takeUntilCorrespondingEvent(final Observable<R> lifecycle,
                                                                       final Function<R, R> correspondingEvents) {
        return Observable.combineLatest(
            lifecycle.take(1).map(correspondingEvents),
            lifecycle.skip(1),
            new BiFunction<R, R, Boolean>() {
                @Override
                public Boolean apply(R bindUntilEvent, R lifecycleEvent) throws Exception {
                    return lifecycleEvent.equals(bindUntilEvent);
                }
            })
            .onErrorReturn(Functions.RESUME_FUNCTION)
            .filter(Functions.SHOULD_COMPLETE);
    }
```

最后我们走到了3，我们一行一行分析：

#### Observable.combineLatest
这行代码实际上是将lifecycle（就是我们传进来的BehaviorSubject）的事件进行了一次分割：

lifecycle.take(1)指的是最近发射的事件，比如说我们在onCreate()中执行了bindToLifecycle，那么lifecycle.take(1)指的就是ActivityEvent.CREATE，经过map(correspondingEvents)，这个map中传的函数就是 1中的ACTIVITY_LIFECYCLE：

```
private static final Function<ActivityEvent, ActivityEvent> ACTIVITY_LIFECYCLE =
        new Function<ActivityEvent, ActivityEvent>() {
            @Override
            public ActivityEvent apply(ActivityEvent lastEvent) throws Exception {
                switch (lastEvent) {
                    case CREATE:
                        return ActivityEvent.DESTROY;
                    case START:
                        return ActivityEvent.STOP;
                    case RESUME:
                        return ActivityEvent.PAUSE;
                    case PAUSE:
                        return ActivityEvent.STOP;
                    case STOP:
                        return ActivityEvent.DESTROY;
                    case DESTROY:
                        throw new OutsideLifecycleException("Cannot bind to Activity lifecycle when outside of it.");
                    default:
                        throw new UnsupportedOperationException("Binding to " + lastEvent + " not yet implemented");
                }
            }
        };
```
也就是说，lifecycle.take(1).map(correspondingEvents)实际上是返回了 **CREATE** 对应的事件 **DESTROY** ,它意味着本次订阅将在Activity的onDestory进行取消。

lifecycle.skip(1)就简单了，除去第一个保留剩下的，以ActivityEvent.Create为例，这里就剩下：

>ActivityEvent.START
>ActivityEvent.RESUME
>ActivityEvent.PAUSE
>ActivityEvent.STOP
>ActivityEvent.DESTROY

第三个参数 意味着，lifecycle.take(1).map(correspondingEvents)的序列和 lifecycle.skip(1)进行combine，形成一个新的序列：

> false,false,fasle,false,true

这意味着，当Activity走到onStart生命周期时，为false,这次订阅不会取消，直到onDestroy，为true，订阅取消。

而后的onErrorReturn和filter是对异常的处理和判断是否应该结束订阅：

```
    //异常处理
    static final Function<Throwable, Boolean> RESUME_FUNCTION = new Function<Throwable, Boolean>() {
        @Override
        public Boolean apply(Throwable throwable) throws Exception {
            if (throwable instanceof OutsideLifecycleException) {
                return true;
            }
            Exceptions.propagate(throwable);
            return false;
        }
    };
    //是否应该取消订阅，可以看到，这依赖于上游的boolean
    static final Predicate<Boolean> SHOULD_COMPLETE = new Predicate<Boolean>() {
        @Override
        public boolean test(Boolean shouldComplete) throws Exception {
            return shouldComplete;
        }
    };
```

#### bind生成LifecycleTransformer

看懂了3，我们回到2，我们生成了一个Observable，然后通过bind(Observable)方法，生成LifecycleTransformer并返回：
```
    public static <T, R> LifecycleTransformer<T> bind(@Nonnull final Observable<R> lifecycle) {
        return new LifecycleTransformer<>(lifecycle);
    }
```

神秘人的神秘面纱就此揭开。


## 总结

RxLifecycle并不难以理解，相反，它的设计思路很简单：

1.在Activity中，定义一个Observable（Subject），在不同的生命周期发射不同的事件；
2.通过compose操作符（内部实际上还是依赖takeUntil操作符），定义了上游数据，当其接收到Subject的特定事件时，取消订阅；
3.Subject的特定事件并非是ActivityEvent，而是简单的boolean，它已经内部通过combineLast操作符进行了对应的转化。

实际上，Subject和ActivityEvent对RxLifecycle的使用者来讲，是对应隐藏的。我们只需要调用它提供给我们的API,而内部的实现者我们无需考虑，但是也只有去阅读和理解了它的思想，我们才能更好的选择使用这个库。

## 转折：AutoDispose
在我沉迷于RxLifecycle对项目的便利时，一个机会，我有幸阅读到了 [Daniel Lew](https://github.com/dlew) 的文章[《Why Not RxLifecycle?》（为什么放弃使用RxLifecycle）](http://blog.danlew.net/2017/08/02/why-not-rxlifecycle/)。

作为**RxLifecycle**的作者，Daniel Lew客观陈述了使用RxLifecycle在开发时所遇到的窘境。并且为我们提供了一个他认为更优秀的设计：

[AutoDispose: Automatic binding+disposal of RxJava 2 streams.](https://github.com/uber/AutoDispose)

我花了一些时间研究了一下**AutoDispose**，不得不承认，**AutoDispose**相比较**RxLifecycle**，前者更加健壮，并且拥有更优秀的拓展性，如果我的项目需要一个处理RxJava自动取消订阅的库，也许**AutoDispose**更为适合。

更让我感到惊喜的是，**AutoDispose**的设计思想，有很大一部分和**RxLifecycle**相似，而且我在其文档上，看到了Uber的工程师对于[Daniel Lew](https://github.com/dlew) 所维护的 [RxLifecycle](https://github.com/trello/RxLifecycle) 感谢，也感谢**Daniel Lew** 对于 **AutoDispose**的很大贡献：

> **以下摘录于AutoDispose的官方文档：**  
>
> Special thanks go to **Dan Lew** (creator of RxLifecycle), who helped pioneer this area for RxJava in android and humored many of the discussions around lifecycle handling over the past couple years that we've learned from. Much of the internal scope resolution mechanics of
**AutoDispose**  are inspired by **RxLifecycle**.

在我尝试将AutoDispose应用在项目中时，我发现国内对于这个库所应用的不多，我更希望有更多朋友能够和我一起使用这个库，于是我准备写一篇关于**AutoDispose**使用方式的文章。

但在这之前，我还是希望能够先将**RxLifecycle**的个人心得分享给大家，原因有二：

一是**RxLifecycle**更简单，原理也更好理解，学习一个库不仅仅是为了会使用它，我们更希望能够从源码中学习到作者优秀的设计和思想。因此，如果是从未接触过这两个库的开发者，直接入手**AutoDispose**，在我看来不如先入手**RxLifecycle**。

二是**AutoDispose**的很大一部分设计核心源于**RxLifecycle**，理解了**RxLifecycle**，更有利于**AutoDispose**的学习。

接下来我将会尝试向大家阐述**AutoDispose**的使用及其内部原理，和RxLifecycle不同，我们不再需要去继承RxActivity/RxFragment，而是内部依赖使用了google不久前推出的架构组件 **Lifecycle**（事实上这也是我青睐AutoDispose的原因之一，尽量避免继承可以给代码更多选择的余地）。

## 2018-3-1追加更新

填坑后归来：

[Android架构中添加AutoDispose解决RxJava内存泄漏](http://blog.csdn.net/mq2553299/article/details/79418068)
