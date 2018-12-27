## 概述
在我的上一篇文章 [《理解RxJava（二）操作符流程原理分析》](http://blog.csdn.net/mq2553299/article/details/78705573) 中，分析了依靠多个操作符链式调用的原理。

简单总结如下：

> 1.**创建**：订阅前，每一步都生成对应的Observable对象，中间的每一步都将上游的Observable存储；
2.**订阅**： 每一步都会生成对应的Observer对上一步生成并存储的Observable进行订阅。订阅的执行顺序是由下到上的。
3.**执行**：先执行每一步传入的函数操作，然后将操作后的数据交给下游的Observer继续处理。 数据的传递和处理顺序是由上到下的。

本文我将尝试对RxJava最核心的 **线程调度** 的原理进行分析。

## 基本代码

来看一下基本代码：
```Java
    @Test
    public void test() throws Exception {
        Observable.create((ObservableOnSubscribe<Integer>) e -> {
            e.onNext(1);
            e.onNext(2);
            e.onComplete();
        }).subscribeOn(Schedulers.io())
          .observeOn(AndroidSchedulers.mainThread())
          .subscribe(i -> System.out.println("onNext : i= " + i));
    }
```
很简单，即订阅时将task交给子线程去做，而数据的回调则在Android主线程中执行。

## 一、subscribeOn()

点击查看源码:

```Java
    public final Observable<T> subscribeOn(Scheduler scheduler) {
        //非空判断和hook
        ObjectHelper.requireNonNull(scheduler, "scheduler is null");
        return RxJavaPlugins.onAssembly(new ObservableSubscribeOn<T>(this, scheduler));
    }
```

有了前两篇文章的基础，我们很清楚，排除掉非空判断和hook相关逻辑，实际上这个方法返回了一个ObservableSubscribeOn对象。

我们有理由猜测这个ObservableSubscribeOn应该和上文的ObservableMap及ObservableDoOnEach相似，都是Observable的一个包装类（装饰器）：

```Java
//1.和上文基本一样，ObservableSubscribeOn也是Observable的一个装饰器
public final class ObservableSubscribeOn<T> extends AbstractObservableWithUpstream<T, T> {
    final Scheduler scheduler;

    public ObservableSubscribeOn(ObservableSource<T> source, Scheduler scheduler) {
       //2.存储上游的ObservableSource和调度器
        super(source);
        this.scheduler = scheduler;
    }

    @Override
    public void subscribeActual(final Observer<? super T> s) {
        //3.new 一个SubscribeOnObserver
        final SubscribeOnObserver<T> parent = new SubscribeOnObserver<T>(s);

        //4.回调方法，这说明下游的onSubscribe回调方法所在线程和线程调度无关
        //  是订阅时所在的线程
        s.onSubscribe(parent);

        //5.立即执行线程调度
        parent.setDisposable(scheduler.scheduleDirect(new SubscribeTask(parent)));
    }
}
```

前两步我们不需要 再多解释，直接看第三点，我们看看SubscribeOnObserver这个类：

### SubscribeOnObserver
```Java
    static final class SubscribeOnObserver<T> extends AtomicReference<Disposable> implements Observer<T>, Disposable {

        private static final long serialVersionUID = 8094547886072529208L;
        //下游的Observer
        final Observer<? super T> actual;
        //保存上游的Disposable，自身dispose时，连同上游一起dispose
        final AtomicReference<Disposable> s;

        SubscribeOnObserver(Observer<? super T> actual) {
            this.actual = actual;
            this.s = new AtomicReference<Disposable>();
        }

        @Override
        public void onSubscribe(Disposable s) {
            DisposableHelper.setOnce(this.s, s);
        }

        @Override
        public void onNext(T t) {
            actual.onNext(t);
        }

        @Override
        public void onError(Throwable t) {
            actual.onError(t);
        }

        @Override
        public void onComplete() {
            actual.onComplete();
        }

        @Override
        public void dispose() {
            DisposableHelper.dispose(s);
            DisposableHelper.dispose(this);
        }
```

类似Observable和ObservableMap，**SubscribeOnObserver**同样是**Disposable**和**Observer**的一个装饰器，提供了对下游数据的传递，以及将task dispose的接口。

第4步我们之前就讲过了，直接看第5步：
```Java
         //5.立即执行线程调度
        parent.setDisposable(scheduler.scheduleDirect(new SubscribeTask(parent)));
```

我们看看SubscribeTask这个类：

### SubscribeTask

```Java
    final class SubscribeTask implements Runnable {
        private final SubscribeOnObserver<T> parent;

        SubscribeTask(SubscribeOnObserver<T> parent) {
            this.parent = parent;
        }

        @Override
        public void run() {
            source.subscribe(parent);
        }
    }
```
难以置信的简单，**SubscribeTask** 仅仅是一个**Runnable** 接口的实现类而已，通过将SubscribeOnObserver作为参数存起来，在run()方法中添加了上游Observable的被订阅事件，就没有了别的操作，

接下来我们看一下scheduler.scheduleDirect(SubscribeTask)中的代码：

```Java
public abstract class Scheduler {
    //...
    public Disposable scheduleDirect(@NonNull Runnable run) {
        return scheduleDirect(run, 0L, TimeUnit.NANOSECONDS);
    }

    public Disposable scheduleDirect(@NonNull Runnable run, long delay, @NonNull TimeUnit unit) {
        // Worker 本身就是Disposable 的实现类
        // 请注意， createWorker()所创建的worker，
        // 实际就是Schdulers.io()所提供的IoScheduler所创建的worker
        final Worker w = createWorker();
        
        //hook相关
        final Runnable decoratedRun = RxJavaPlugins.onSchedule(run);

        DisposeTask task = new DisposeTask(decoratedRun, w);
        
        //即 worker.schedule(task, 0, TimeUnit.NANOSECONDS): 立即执行task
        w.schedule(task, delay, unit);

        return task;
    }
    //...
}
```
我们不要追究过深，我们看一下这个createWorker方法的注释说明：
```
    /**
     * Retrieves or creates a new {@link Scheduler.Worker} that represents serial execution of actions.
     * 检索或创建一个新的{@link Scheduler.Worker}表示一系列的action
     * 
     * When work is completed it should be unsubscribed using {@link Scheduler.Worker#dispose()}.
     * 当work完成后，应使用{@link Scheduler.Worker＃dispose（）}取消订阅。
     * 
     * Work on a {@link Scheduler.Worker} is guaranteed to be sequential.
     *  {@link Scheduler.Worker} 上面的work保证是顺序执行的
     */
```
现在我们知道了：

> 我们通过调用subscribeOn()传入Scheduler，当下游ObservableSource被订阅时（请注意，订阅顺序是由下到上的），距离最近的线程调度subscribeOn()方法中，保存的Scheduler会**创建一个**worker(对应**相应的线程**，本文中为IoScheduler)，在其对应的线程中，**立即执行task**

关于对应的Worker相关和IoScheduler相关，篇幅所限，本文不做细讲，有兴趣的同学可以自行研究。

### 多次subscribeOn()

现在考虑一个问题，假如在我们的代码中，多次使用了subscribeOn()代码，到线程会怎么处理呢？

> 上文已经讲到了，不管我们怎么通过subscribeOn()方法切换线程，由于**订阅执行顺序是由下到上**，因此当最上游的ObservableSource被订阅时，所在线程当然是**距离上游最近的subscribeOn()所提供的线程**，即**最终Observable总是在第一个subscribeOn()所在的线程中执行**。


## 二、observeOn()

先看observeOn()内部，果然是hook+Observable的包装类：

```
    public final Observable<T> observeOn(Scheduler scheduler) {
        return observeOn(scheduler, false, bufferSize());
    }

    public final Observable<T> observeOn(Scheduler scheduler, boolean delayError, int bufferSize) {
        ObjectHelper.requireNonNull(scheduler, "scheduler is null");
        ObjectHelper.verifyPositive(bufferSize, "bufferSize");

        //实例化ObservableObserveOn对象并返回
        return RxJavaPlugins.onAssembly(new ObservableObserveOn<T>(this, scheduler, delayError, bufferSize));
    }
```

再看ObservableObserveOn：

```
public final class ObservableObserveOn<T> extends AbstractObservableWithUpstream<T, T> {
    final Scheduler scheduler;
    final boolean delayError;
    final int bufferSize;
    public ObservableObserveOn(ObservableSource<T> source, Scheduler scheduler, boolean delayError, int bufferSize) {
        super(source);
        //1.相关依赖注入
        this.scheduler = scheduler;
        this.delayError = delayError;
        this.bufferSize = bufferSize;
    }

    @Override
    protected void subscribeActual(Observer<? super T> observer) {
        if (scheduler instanceof TrampolineScheduler) {
            source.subscribe(observer);
        } else {
            //2.创建主线程的worker
            Scheduler.Worker w = scheduler.createWorker();
            //3.上游数据源被订阅
            source.subscribe(new ObserveOnObserver<T>(observer, w, delayError, bufferSize));
        }
    }
}
```
和subscribeOn()不同的是，我们并不是立即在对应的线程执行task，而是将对应的线程(实际上是worker)作为参数，实例化ObserveOnObserver并存储起来。

当上游的数据传递过来时，ObserveOnObserver执行对应的方法，比如onNext(T)，再切换到对应线程中，并交由下游的Observer去接收：

### ObserveOnObserver

ObserveOnObserver中代码极多，我们简单了解原理后，以onNext（T）为例：

```
static final class ObserveOnObserver<T> extends BasicIntQueueDisposable<T>
    implements Observer<T>, Runnable {

        //...省略其他代码
        ObserveOnObserver(Observer<? super T> actual, Scheduler.Worker worker, boolean delayError, int bufferSize) {
            this.actual = actual;
            this.worker = worker;
            this.delayError = delayError;
            this.bufferSize = bufferSize;
        }
        //队列
        SimpleQueue<T> queue;

       @Override
        public void onNext(T t) {
            if (done) {
                return;
            }
            //将数据存入队列
            if (sourceMode != QueueDisposable.ASYNC) {
                queue.offer(t);
            } 
            //对应线程取出数据并交由下游的Observer
            schedule();
        }

        void schedule() {
            if (getAndIncrement() == 0) {
                worker.schedule(this);
            }
        }
         //...省略其他代码
}
```

###　多次observerOn()

> 由上文得知，与subscribeOn()相反，observerOn()操作会将切换到对应的线程，然后交由下游的Observer处理，因此**observerOn()仅对下游的Observer生效**，并且，**如果多次调用**，observerOn()的线程调度**会持续到下一个observerOn()操作之前**。

我在[《RxJava(11-线程调度Scheduler)》 @open-Xu](http://blog.csdn.net/xmxkf/article/details/51821940)文章中看到了这张图片，完美诠释了observerOn()的原理：

![observerOn()的原理](http://upload-images.jianshu.io/upload_images/7293029-0ce2581de41b338a?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

### 注意：
接下来的内容都是[@open-Xu](http://blog.csdn.net/u010163442)在他的文章[《RxJava(11-线程调度Scheduler)》](http://blog.csdn.net/xmxkf/article/details/51821940)中总结的笔记，为了方便以后翻阅，特此转载过来，下文的转载请@原作者[open-Xu](http://blog.csdn.net/u010163442)并注明原文地址，谢谢！

## 调度器的种类

| 调度器类型                                |效果                 |
| -------------                                                           | -------------             |
|Schedulers.computation( )                        |用于计算任务，如事件循环或和回调处理，不要用于IO操作(IO操作请使用Schedulers.io())；默认线程数等于处理器的数量      |
|Schedulers.from(executor)                      |使用指定的Executor作为调度器
|
|Schedulers.immediate( )                      |在当前线程立即开始执行任务     |
|Schedulers.newThread( )                    |为每个任务创建一个新线程     |
|Schedulers.io( )                     |用于IO密集型任务，如异步阻塞IO操作，这个调度器的线程池会根据需要增长；对于普通的计算任务，请使用Schedulers.computation()；Schedulers.io( )默认是一个CachedThreadScheduler，很像一个有线程缓存的新线程调度器      |
|Schedulers.trampoline( )                     |	当其它排队的任务完成后，在当前线程排队开始执行     |
|AndroidSchedulers.mainThread( )                   |主线程，UI线程，可以用于更新界面     |

## 总结

#### subscribeOn()
* 订阅顺序当从下到上，上游的ObservableSource被订阅时，**先切换线程**，然后**立即执行task**;
* 当存在**多个subscribeOn()**方法时，**仅第一个subscribeOn()有效**。

#### observerOn()
*  订阅顺序当从下到上，上游的ObservableSource被订阅时，会将对应的**worker**创建并作为构造参数存储在**Observer的装饰器**中，并**不会立即切换线程**；
* 当数据由上游发送过来时，先将数据存储到**队列**中，然后**切换线程**，然后在新的线程中将数据发送给**下游的Observer**；
* 当存在**多个observerOn()**方法时，仅对距下游**下一个observerOn()之前**的observer有效

### 参考文章

1.[《RxJava(11-线程调度Scheduler)》 @open-Xu](http://blog.csdn.net/xmxkf/article/details/51821940)

2.[《RxJava2 源码解析（二）》@张旭童](http://www.jianshu.com/p/6ef45f8ee79d)
