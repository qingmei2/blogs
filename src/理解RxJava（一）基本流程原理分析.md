
> 最近一直没有机会，好好写博客，可能还是太浮躁了，自己对自己的这种状态也不是特别满意。近几日准备安下心来，好好研究一下RxJava，把这期间的所得总结成一个系列，尽量都写博客中，看看这个阶段结束时自己能达到怎么样的程度。

## 概述
在日常的Android项目开发中，RxJava+Retrofit是一个万金油网络请求框架，通常情况下，我们的代码大概是这样的：

```Java
 public void requestUserInfo(String userName) {
	serviceManager.getUserInfoService()
	              .getUserInfo(userName)
	              .subscribeOn(Schedulers.io())
	              .observeOn(AndroidSchedulers.mainThread())
	              .subscribe(info -> {
					  //do something
				  });
 }
```

事实上，我很少去关心RxJava内部是如何实现的，因为这样我也能够轻松定制项目中的网络请求需求，并且写出看起来还不错的代码。

但是很快我就事与愿违了，因为在测试RxJava相关的业务代码时，面对需要测试的代码块，我束手无策，我接连踩到了很多坑。虽然最后我都通过万能的google一一解决了，但是我已经不满足现状，我可以轻易提出很多个让我疑惑的问题：

> 1 RxJavaPlugin是干什么的
> 2 链式调用中当存在数个Observable.subscribeOn(）时的处理方式
> 3 链式调用中当存在数个Observable.observeOn(）时的处理方式
> 4 数据是如何经过操作符进行的处理
> 5 各个doOnXXX（）方法回调所在的线程？

等等......

于是我就有了这个念头：研究源码，彻底将它整明白。

### 源码版本

**RxJava官方Github**: https://github.com/ReactiveX/RxJava

**Branch**:2.x (RxJava2)   

**版本号**：2.1.3

## 一.基本流程

 我决定从简单的代码入手分析RxJava2:

```Java
public class RxJavaTest {

    @Test
    public void test() throws Exception {
        Observable.create(new ObservableOnSubscribe<String>() {
            @Override
            public void subscribe(ObservableEmitter<String> e) throws Exception {
                e.onNext(1);
                e.onNext(2);
                e.onComplete();
             }
        }).subscribe(new Observer<Integer>() {
            @Override
            public void onSubscribe(Disposable d) {}

            @Override
            public void onNext(Integer integer) {
                System.out.println(String.valueOf(integer));
            }

            @Override
            public void onError(Throwable e) {}

            @Override
            public void onComplete() {}
        });
    }
}
```
代码很简单，我们可以简单分成两步：

> 1.Observable.create()
> 2.Observable.subscribe()

一步步进行分析。

## 二.Observable.create()

我们点击进入Observable.create()方法的内部：

```Java
    public static <T> Observable<T> create(ObservableOnSubscribe<T> source) {
        //...（表示排除非关键代码，下同）
        return RxJavaPlugins.onAssembly(new ObservableCreate<T>(source));
    }
```

可以看到，create方法的返回值仍然是一个Observable，这是当然的，我们看到RxJavaPlugins.onAssembly():

```Java

/**
 * Calls the associated hook function.
 */
 public static <T> Observable<T> onAssembly(@NonNull Observable<T> source) {
    //...
 }
```
这是一个hook方法，我们暂时先忽略它，默认这个方法就是返回了直接返回source,即传入的参数对象，那么实际上，create方法的关键就是下面这行代码：
```Java
 return new ObservableCreate<T>(source);
```

来看看这个ObservableCreate是何方神圣：

```Java

//1.ObservableCreate是Observable的子类
public final class ObservableCreate<T> extends Observable<T> {

    final ObservableOnSubscribe<T> source;
    
    //2.注意这里，实际上Observable.create方法执行了下面的代码
    public ObservableCreate(ObservableOnSubscribe<T> source) {
        this.source = source;
    }
    
    //3.subscribeActual方法是Observable.subscribe()后才执行的
    @Override
    protected void subscribeActual(Observer<? super T> observer) {
        CreateEmitter<T> parent = new CreateEmitter<T>(observer);
        observer.onSubscribe(parent);

        try {
            source.subscribe(parent);
        } catch (Throwable ex) {
            Exceptions.throwIfFatal(ex);
            parent.onError(ex);
        }
    }
}
```
我已经把重点标记了，可以得知：
* 1.ObservableCreate本身就是Observable的子类，因此Observable.create()实际上就是创建并返回一个ObservableCreate对象；
* 2.在ObservableCreate的构造函数中，实际上是将create()方法中的参数ObservableOnSubscribe依赖注入ObservableCreate中并保存；
* 3.subscribeActual()方法是在Observable.subscribe()订阅时执行的，我们先知道这点，一会再详细介绍它。

现在关于ObservableCreate这个类本身我们基本都了解了，我们来看看我们传进来的ObservableOnSubscribe是什么：

```Java
public interface ObservableOnSubscribe<T> {

    /**
     * Called for each Observer that subscribes.
     */
    void subscribe(@NonNull ObservableEmitter<T> e) throws Exception;
}
```
原来ObservableOnSubscribe只是一个接口，它就是我们传进来的这个：

```Java

Observable.create(new ObservableOnSubscribe<String>() {
            //就是它
            @Override
            public void subscribe(ObservableEmitter<String> e) throws Exception {
                e.onNext(1);
                e.onNext(2);
                e.onComplete();
             }
        }).subscribe(...);
```

实际上，ObservableOnSubscribe就是让我们去实现的接口，我们将要发射的事件都通过ObservableEmitter发射器对象进行发射（本文中是依次发射Integer 1、2），可以说，在本文的案例中，ObservableOnSubscribe就是数据流的源头。

### 整理思路

现在我们已知的情报，在Observable调用create()方法时：

* 1.我们实现了ObservableOnSubscribe接口，并在接口的subscribe方法中，传入了数据源，这些数据源需要通过ObservableEmitter进行发射。
* 2.在create()方法中，抛开hook不谈，实际上方法内部是创建了一个ObservableCreate对象并返回，这个ObservableCreate类本身就是Observable的子类
* 3.在ObservableCreate的构造方法中，将我们实现的ObservableOnSubscribe（定义事件的发射顺序）作为成员保存在ObservableCreate中。

现在基本已经清楚了，但是我们还有两个问题需要解答：

> 1.ObservableEmitter发射器对象何时实例化并发射数据？
> 2.接下来如何才能实现Observable的订阅？

## 三.Observable.subscribe()

我们点击进入查看Observable.subscribe()源码：

```Java
public final void subscribe(Observer<? super T> observer) {
        //...
        try {
            //...
            //1.我们只需要关注下面这行代码
            subscribeActual(observer);
        } catch (NullPointerException e) {
            throw e;
        } catch (Throwable e) { 
            //...
        }
    }
```

抛开非核心代码和hook相关代码，我们可以看到，当我们调用 Observable.subscribe(observer)方法时，实际上就是调用Observable.subscribeActual(observer)方法。

应该清楚的记得，在本文中，此时执行subscribe()方法的Observable，实际上应该就是ObservableCreate对象。

因此，Observable.subscribe(observer)，实际上执行的是下面的代码：

```Java
public final class ObservableCreate<T> extends Observable<T> {
    
    final ObservableOnSubscribe<T> source;
    
    public ObservableCreate(ObservableOnSubscribe<T> source) {
        this.source = source;
    }
    
    //真正执行的代码
    @Override
    protected void subscribeActual(Observer<? super T> observer) {
        //1.请注意，参数中的observer实际上就是我们要打印onNext的observer
        //2.实例化CreateEmitter对象
        CreateEmitter<T> parent = new CreateEmitter<T>(observer);
        
        //3.执行observer的onSubscribe方法，这里可以看到：
        //  onSubscribe()回调所在的线程是ObservableCreate执行subscribe()所在的线程
        //  和subscribeOn()/observeOn()无关！
        observer.onSubscribe(parent);

        try {
            //4.ObservableOnSubscribe执行subscribe()方法
            //这个方法就是我们在create()中定义的代码
            //其实就是发射数据源，parent是CreateEmitter
            source.subscribe(parent);
        } catch (Throwable ex) {
        
            //5.异常处理
            Exceptions.throwIfFatal(ex);
            parent.onError(ex);
        }
    }
}
```

### 整理思路

我把需要解释的基本都放到了注释中，简单总结一下流程：

* 1.在Observable执行subscribe方法时，实际上是执行了ObservableCreate.subscribeActual()方法，并将我们打印数据的回调Observer作为参数传了进去 
* 2.在subscribeActual方法内部，首先示例化了一个CreateEmitter发射器对象，然后执行Observer的onSubscribe()回调。
* 3.之后，就是执行一开始我们在代码中声明的ObservableOnSubscribe对象的subscribe方法，开始通过CreateEmitter发射器发射数据,即执行：

```Java
public void subscribe(ObservableEmitter<String> e) throws Exception {
                e.onNext(1);
                e.onNext(2);
                e.onComplete();
}
```

现在的问题是，仅仅通过发射器ObservableEmitter.onNext(1)，我们需要的打印代码哪里去了？
```Java
Observable.create(//...)
        .subscribe(new Observer<Integer>() {
        
            //...
            
            @Override
            public void onNext(Integer integer) {
                System.out.println(String.valueOf(integer));
            }
            
            //...
        
            
        });
```

我们拐回到这行代码：
``` Java
protected void subscribeActual(Observer<? super T> observer) {
    //...
    
    //2.实例化CreateEmitter对象
    CreateEmitter<T> parent = new CreateEmitter<T>(observer);

    //...
}
```
我们来看看CreateEmitter这个类：

```Java
static final class CreateEmitter<T>
    extends AtomicReference<Disposable>
    
    // 1.请注意，CreateEmitter实现了Disposable接口
    // 这也就说明了在执行上面subscribeActual()步骤3 observer.onSubscribe()的回调时，可以将ObservableEmitter作为参数传入了
    // ObservableEmitter也是Disposable
    
    implements ObservableEmitter<T>, Disposable {


        private static final long serialVersionUID = -3434801548987643227L;

        final Observer<? super T> observer;
        
        //2.初始化时，会将打印数据的回调Observer作为成员储存
        CreateEmitter(Observer<? super T> observer) {
            this.observer = observer;
        }
        
        @Override
        public void onNext(T t) {
            if (t == null) {
                onError(new NullPointerException("onNext called with null. Null values are generally not allowed in 2.x operators and sources."));
                return;
            }
            if (!isDisposed()) {
                //3.在ObservableEmitter.onNext()发射数据时，实际上调用了observer的onNext()回调打印数据
                observer.onNext(t);
            }
        }
        
        // 4(额外):当执行onError()时，会执行dispose()方法取消订阅
        @Override
        public void onError(Throwable t) {
            if (!tryOnError(t)) {
                RxJavaPlugins.onError(t);
            }
        }

        @Override
        public boolean tryOnError(Throwable t) {
            if (t == null) {
                t = new NullPointerException("onError called with null. Null values are generally not allowed in 2.x operators and sources.");
            }
            if (!isDisposed()) {
                try {
                    observer.onError(t);
                } finally {
                    dispose();
                }
                return true;
            }
            return false;
        }
        
        // 5(额外):当执行onError()时，会执行dispose()方法取消订阅
        //这也就说明为什么 onComplete()和onError()方法只能触发其中的一个了
        @Override
        public void onComplete() {
            if (!isDisposed()) {
                try {
                    observer.onComplete();
                } finally {
                    dispose();
                }
            }
        }

        @Override
        public void setDisposable(Disposable d) {
            DisposableHelper.set(this, d);
        }

        @Override
        public void setCancellable(Cancellable c) {
            setDisposable(new CancellableDisposable(c));
        }

        @Override
        public ObservableEmitter<T> serialize() {
            return new SerializedEmitter<T>(this);
        }

        @Override
        public void dispose() {
            DisposableHelper.dispose(this);
        }

        @Override
        public boolean isDisposed() {
            return DisposableHelper.isDisposed(get());
        }
    }
```

基本都写到了注释中，应该不难理解。

## 总结

分析一下各个类的职责：

* Observable ：个人理解是装饰器模式下的基类，实际上所有操作都是Observable的子类进行的实现
* ObservableOnSubscribe: 接口，定义了数据源的发射行为
* ObservableCreate: 装饰器模式的具体体现，内部存储了数据源的发射事件，和subscribe订阅事件
* ObservableEmitter： 数据源发射器，内部存储了observer
* Observer: 接收到数据源后的回调（比如打印数据等）

简单总结一下：

1.Observable.create()，实例化ObservableCreate和ObservableOnSubscribe，并存储数据源发射行为，准备发射（我已经准备好数据源，等待被订阅）
2.Observable.subscribe(),实例化ObservableEmitter（发射器ObservableEmitter准备好！数据发射后，数据处理方式Observer已准备好！）
3.执行Observer.onSubscribe()回调，ObservableEmitter作为Disposable参数传入
4.执行ObservableOnSubscribe.subscribe()方法  （ObservableEmitter发射数据，ObservableEmitter内部的Observer处理数据）

此外我们还可以得到的：
> 5.CreateEmitter 中，只有Observable和Observer的关系没有被dispose，才会回调Observer的onXXXX()方法。

> 6.Observer的onComplete()和onError() 互斥只能执行一次，因为CreateEmitter在回调他们两中任意一个后，都会自动dispose()。

> 7.onSubscribe()是在我们执行subscribe()这句代码的那个线程回调的，并不受线程调度影响。

## 参考

1.RxJava2 源码解析（一）@张旭童 
http://www.jianshu.com/p/23c38a4ed360

2.RxJava2 源码解析——流程 @Robin_Lrange 
http://www.jianshu.com/p/e5be2fa8701c
