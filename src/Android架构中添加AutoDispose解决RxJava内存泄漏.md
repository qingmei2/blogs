## 概述
在我的上一篇文章[ 解决RxJava内存泄漏（前篇）：RxLifecycle详解及原理分析](https://blog.csdn.net/mq2553299/article/details/78927617) 中，详细阐述了 

* 如何通过使用 [RxLifecycle](https://github.com/trello/RxLifecycle) 解决Android开发中RxJava的可能会导致的**内存泄漏**问题；
* [RxLifecycle](https://github.com/trello/RxLifecycle) 内部的实现原理;
 
在文章的最后，我提到了 [AutoDispose](https://github.com/uber/AutoDispose) 这个库，这个库同样可以解决Android生命周期组件导致的RxJava的内存泄漏情况。

但是不得不考虑的是，目前国内的Android开发圈子中，RxLifecycle已经逐渐为人所熟知，包括著名的一些开源架构也都采用RxLifecycle替代手动的处理方式 。比如 [JessYan](https://github.com/JessYanCoding) 的 [MVPArms](https://github.com/JessYanCoding/MVPArms) 。

随着越来越多的项目添加了RxLifecycle，我需要给出一个足够有说服力的理由，足够让工程师同僚们去尝试将AutoDispose替换RxLifecycle，或者让他们在新的项目中优先考虑使用AutoDispose。  

我花了一些时间研究了AutoDispose源码，并且在公司的新项目中尝试使用AutoDispose，我尝试将自己的所得分享出来，希望能够抛砖引玉——如果看完之后，仍旧觉得 **AutoDispose** 的思想和设计没什么用，至少也让客官您斩掉这个念头，继续安稳地使用 **RxLifecycle** 。

本文的主要内容如下：

*  AutoDispose的基础使用
*  AutoDispose的基本原理
*  AutoDispose和RxLifecycle的区别
*  如何添加到目前的Android项目中（以MVP架构为例）
* 小结 

## 基础使用
**官方文档永远是最好的说明书：**

[AutoDispose: Automatic binding+disposal of RxJava 2 streams. ](https://github.com/uber/AutoDispose)
 
###  1、添加依赖
 
```
compile 'com.uber.autodispose:autodispose:x.y.z'
compile 'com.uber.autodispose:autodispose-android-archcomponents:x.y.z'
```
x.y.z请参照官网的最新版本，写这篇文章时，最新版本为0.6.1,即：
```
compile 'com.uber.autodispose:autodispose:0.6.1'
compile 'com.uber.autodispose:autodispose-android-archcomponents:0.6.1'
```

 2、在你的代码中直接使用,比如：
```java
//在Activity中使用
myObservable
    .map(...)
    .subscribeOn(...)
    .observeOn(...)
    .as(AutoDispose.autoDisposable(AndroidLifecycleScopeProvider.from(this))  
    .subscribe(s -> ...);
```
通过给Observable（或者Single、Completable、Flowable等响应式数据类型，本文皆以Observable为例）添加这行代码：

> as(AutoDispose.autoDisposable(AndroidLifecycleScopeProvider.from(lifecycleOwner))
 
该Observable就会在Activity的onDestory()生命周期时，自动解除订阅，以防止因生命周期组件的生命周期而导致的RxJava内存泄漏事件。
 
 看到这里，对比RxLifecycle的代码使用方式：

```java
myObservable
    .compose(RxLifecycle.bind(lifecycle))
    .subscribe();
```
 
 似乎都差不多，都是通过添加一行代码实现RxJava事件流与组件生命周期的绑定，那么替换RxLifecycle的意义何在？

先按住这个问题不表，我先简单阐述一下AutoDispose的原理。

## 基本原理
 
**直接长篇大论针对源码逐行分析，无疑是愚蠢的**——我无法保证读者能够耐住性子看完枯燥无味的源码分析，然后兴致勃勃去尝试这个库，这与我初衷不符。
 
 抛开细节，直接阐述其设计思想。

首先前提是，您需要对Google最新的Lifecycle组件有一定的了解，在这里我列出之前写的博客以供参考：

[Android官方架构组件:Lifecycle详解&原理分析](http://blog.csdn.net/mq2553299/article/details/79029657)
 
 同时，如果您对RxLifecycle的原理也掌握的话，相信对于接下来的内容，阅读起来会更加轻松：

[ 解决RxJava内存泄漏（前篇）：RxLifecycle详解及原理分析](blog.csdn.net/mq2553299/article/details/78927617) 

我们首先思考三个问题：

> as(AutoDispose.autoDisposable(AndroidLifecycleScopeProvider.from(this))

*  this是一个什么样的参数？
*  如何实现生命周期的绑定？
*  as方法执行后生成了一个什么？
 

### 1、我传了一个什么参数？
 
 这个this直观的讲，就是Activity本身，当然它也可以是Fragment，这个参数对象只有一个要求，就是必须实现**LifecycleOwner**接口。

LifecycleOwner接口是Google Android官方架构组件:Lifecycle的一个重要组件，在v7包中，FragmentActivity和Fragment都实现了这个接口，实现了这个接口的对象都拥有生命周期（Lifecycle）。

这意味着，不仅是AppCompatActiviy（FragmentActivity的子类）和Fragment，只要是实现了LifecycleOwner的类，都可以作为参数传给AutoDispose，用以控制Observable和组件生命周期的绑定。

### 2、如何实现生命周期的绑定

参考RxLifecycle的原理：

>1.在Activity中，定义一个Observable（Subject），在不同的生命周期发射不同的事件； 
2.通过compose操作符（内部实际上还是依赖takeUntil操作符），定义了上游数据，当其接收到Subject的特定事件时，取消订阅； 
3.Subject的特定事件并非是ActivityEvent，而是简单的boolean，它已经内部通过combineLast操作符进行了对应的转化。

 AutoDispose获取了Activity(LifecycleOwner)对象，并定义了一个新的Observable，在Activity的不同生命周期中，发射对应的事件。
 
和RxLifecycle很类似的是，AutoDispose在被订阅时，获取到Activity当前的生命周期，并找到对应需要结束订阅的生命周期事件：

```
  private static final Function<Lifecycle.Event, Lifecycle.Event> DEFAULT_CORRESPONDING_EVENTS =
      new Function<Lifecycle.Event, Lifecycle.Event>() {
        @Override public Lifecycle.Event apply(Lifecycle.Event lastEvent) throws Exception {
          switch (lastEvent) {
            case ON_CREATE:
              return Lifecycle.Event.ON_DESTROY;//比如我在onCreate到onStart之间订阅，我会在ON_DESTROY时结束订阅
            case ON_START:
              return Lifecycle.Event.ON_STOP;
            case ON_RESUME:
              return Lifecycle.Event.ON_PAUSE;
            case ON_PAUSE:
              return Lifecycle.Event.ON_STOP;
            case ON_STOP://如果是onPause之后订阅，会抛出异常
            case ON_DESTROY:
            default:
              throw new LifecycleEndedException("Lifecycle has ended! Last event was " + lastEvent);
          }
        }
      };
```
 
 也就是说，在我们的ObservableA订阅时，就已经知道了自己在Activity的哪个生命周期让AutoDispose内部自定义的ObservableB自动发射事件，ObservableA监听到这个事件时且未dispose，解除订阅避免内存泄漏。

毕竟内存泄漏是少数，更大的可能是ObservableA早就执行完任务dispose了，因此ObservableB实际上就是一个Maybe，类似于

> ObservableA.takeUntil( Maybe< true > )

下面为核心代码：
 
```
 public static <E> Maybe<LifecycleEndNotification> resolveScopeFromLifecycle(
      Observable<E> lifecycle,
      final E endEvent) {
    return lifecycle.skip(1)
        .map(new Function<E, Boolean>() {
          @Override public Boolean apply(E e) throws Exception {
            return e.equals(endEvent);//是否是要解除订阅的生命周期事件
          }
        })
        .filter(IDENTITY_BOOLEAN_PREDICATE)//筛选为true的数据，即筛选解除订阅的生命周期事件
        .map(TRANSFORM_TO_END)//转换为LifecycleEndNotification.INSTANCE，这个枚举只用来通知解除订阅
        .firstElement();//返回值为Maybe，因为可能不到对应生命周期，ObservableA就已经完成任务，onComplete() -> dispose了
  }
	
  private static final Function<Object, LifecycleEndNotification> TRANSFORM_TO_END =
      new Function<Object, LifecycleEndNotification>() {
        @Override public LifecycleEndNotification apply(Object o) throws Exception {
          return LifecycleEndNotification.INSTANCE;
        }
      };

  public enum LifecycleEndNotification {
    INSTANCE
  }
```
 
### 3、as方法执行后生成了一个什么？

as方法内部生成了一个AutoDisposeConverter对象，类似于compose,不同的是，Observable通过compose生成的对象还是Observable，但as方法生成的则是一个新的对象：

```
public final <R> R as(@NonNull ObservableConverter<T, ? extends R> converter)
```
 实际上，抛开复杂的细节，AutoDispose最终将原来的Observable，生成了一个新的AutoDisposeObservable对象, 在执行订阅时，也生成了一个新的AutoDisposingObserverImpl对象，篇幅所限，不再细述。

 
 
## AutoDispose和RxLifecycle的区别
 
 从上面的原理来讲，似乎AutoDispose和RxLifecycle两者没什么区别，原理都极为相似。

事实确实如此，因为AutoDispose本身就是很大一部分借鉴了RxLifecycle，同时，RxLifecycle的作者Daniel Lew 对于 AutoDispose的开发也有很多帮助：

> 以下摘录于AutoDispose的官方文档：
> 
Special thanks go to Dan Lew (creator of RxLifecycle), who helped pioneer this area for RxJava in android and humored many of the discussions around lifecycle handling over the past couple years that we’ve learned from. Much of the internal scope resolution mechanics of 
AutoDispose are inspired by RxLifecycle.

那么，究竟是什么原因，让RxLifecycle的作者Daniel Lew 在他自己的文章中，罗列出RxLifecycle在开发中的窘境，并同时强烈推荐使用AutoDispose呢？

>  I'm going to keep maintaining RxLifecycle because people are still using it (including Trello), but in the long term I'm pulling away from it. For those still wanting this sort of library, I would suggest people look into AutoDispose, since I think it is better architecturally than RxLifecycle. 
(我将会继续维护RxLifecycle因为很多人都在使用它，但是我会渐渐放弃这个库，我建议大家可以参考AutoDispose，因为我认为它的设计更加优秀 )
  

### 压倒性优势?
 
 我们来看看RxLifecycle的局限性：

#### 1、需要继承父类（RxActivity / RxFragment等）
> 对于设计来讲，【组合】的灵活度大多数情况都优于【继承】，而RxLifecycle在父类中声明了一个PublishSubject，用来发射生命周期事件，这是导致其局限性的原因之一。 

#### 2、如何处处绑定生命周期？

 最简单的例子，我的RecyclerView的Adapter中订阅了Observable，亦或者，在MVP的架构或者MVVM架构中，我的presenter或者我的viewModel无法直接获取RxActivity的引用(作为View层，更应该抽象为一个接口与Presenter进行交互)。
 
这意味着，想要进行Observable的生命周期绑定，在RecyclerView的Adapter中,我必须要通过将Activity作为依赖，注入到Adapter中：

```
new ListAdapter(RxActivity activity);
```

而对于Presenter，我需要对View抽象接口进行instanceof 的判断：

```
if (view instanceof RxActivity) {
   return bindToLifecycle((RxActivity) view);
}
```
当然还有一些其他的情况，请参考原作者的文章：

[Why Not RxLifecycle?
](http://blog.danlew.net/2017/08/02/why-not-rxlifecycle/)
 
 
## 如何添加到目前的Android项目中（以MVP架构为例）

AutoDispose正常直接在项目中使用没有什么问题，比如：

```java
//在Activity中使用
myObservable
	.as(AutoDispose.autoDisposable(AndroidLifecycleScopeProvider.from(this))  
    .subscribe(s -> ...);
```
 
 但是，在实际的生产环境中，我们更希望有一个良好的方式，能够达到统一管理Observable绑定生命周期的效果，在此笔者以MVP的架构为例，抛砖引玉，希望能够获得大家的意见和建议。

### 1、封装Util类

首先封装Util类，将职责赋予RxLifecycleUtils：

```Java
public class RxLifecycleUtils {

    private RxLifecycleUtils() {
        throw new IllegalStateException("Can't instance the RxLifecycleUtils");
    }

    public static <T> AutoDisposeConverter<T> bindLifecycle(LifecycleOwner lifecycleOwner) {
        return AutoDispose.autoDisposable(
                AndroidLifecycleScopeProvider.from(lifecycleOwner)
        );
    }
}
``` 

### 2、面向LifecycleOwner接口


 现在，只要持有LifecycleOwner对象，Observable都可以通过RxLifecycleUtils.bindLifecycle(LifecycleOwner)进行绑定。

比如我们在BaseActivity/BaseFragment中添加代码：

```
public abstract class BaseActivity extends AppCompatActivity implements IActivity {
	//...忽略其他细节
	protected <T> AutoDisposeConverter<T> bindLifecycle() {
        return RxLifecycleUtils.bindLifecycle(this);
    }
}
```
这样，在任何BaseActivity的实现类中，我们都可以通过下述代码实现Observable的生命周期绑定：

```
myObservable
	.as(bindLifecycle())  
    .subscribe(s -> ...);
```
 
 但是这样的行为意义不大，我们更希望在Presenter中也能够像上述代码一样达到Observable生命周期的绑定，但是不持有Activity的对象的同时，避免RxLifecycle中尴尬的instanceof判断。

可行吗，可行。

### 3、使用Google官方Lifecycle组件

首先让我们的IPresenter接口实现LifecycleObserver接口：

```
public interface IPresenter extends LifecycleObserver {

    @OnLifecycleEvent(Lifecycle.Event.ON_CREATE)
    void onCreate(@NotNull LifecycleOwner owner);
    
    @OnLifecycleEvent(Lifecycle.Event.ON_DESTROY)
    void onDestroy(@NotNull LifecycleOwner owner);

    @OnLifecycleEvent(Lifecycle.Event.ON_ANY)
    void onLifecycleChanged(@NotNull LifecycleOwner owner,
                            @NotNull Lifecycle.Event event);
}
```
然后在BasePresenter中管理LifecycleOwner:
```
public class BasePresenter<V extends IView, M extends IModel> implements IPresenter {

    @Getter
    protected V mRootView;

    @Getter
    protected M mModel;

    private LifecycleOwner lifecycleOwner;

    public BasePresenter(V rootView, M model) {
        this.mRootView = rootView;
        this.mModel = model;
    }

    protected <T> AutoDisposeConverter<T> bindLifecycle() {
        if (null == lifecycleOwner)
            throw new NullPointerException("lifecycleOwner == null");
        return RxLifecycleUtils.bindLifecycle(lifecycleOwner);
    
    public void onCreate(@NotNull LifecycleOwner owner) {
        this.lifecycleOwner = owner;
    }
    
    public void onDestroy(@NotNull LifecycleOwner owner) {
            if (mModel != null) {
                mModel.onDestroy();
                this.mModel = null;
            }
            this.mRootView = null;
        }
    }
```    
最后在Activity中添加Presenter为观察者，观察Activity的生命周期
```
public abstract class BaseActivity<P extends IPresenter> extends AppCompatActivity implements IActivity {
	
	@Inject
	protected P presenter;

	    @Override
    protected void onCreate(@Nullable Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(getLayoutId());
        //...忽略其他代码，比如ButterKnife、Dagger2等
		lifecycle.addObserver(presenter);
    }	    

	protected <T> AutoDisposeConverter<T> bindLifecycle() {
        return RxLifecycleUtils.bindLifecycle(this);
    }
}
```    
这样，我们即使在Presenter中，也能任意使用myObservable.as(bindLifecycle()) 方法了，和RxLifecycle相比，更加灵活。
    
篇幅所限，仅以最简单的设计思路进行阐述，更多情况的使用请在项目中自行拓展。
    
更详细的使用，请参考我的个人MVP架构：[MvpArchitecture-Android](https://github.com/qingmei2/MvpArchitecture-Android)
    
### 2018.9追加

时隔数月，我的个人MVP架构：[MvpArchitecture-Android](https://github.com/qingmei2/MvpArchitecture-Android) 已经无法满足我个人开发的需求，于我而言它更像是一个demo,因此我停止了对它的维护。

相比前者，我更喜欢`MVVM`架构中`AutoDispose`的这次实践，它更趋近我理想中的设计，更加`Reactive`和`Functional`：

 > [MVVM-Rhine:The MVVM Architecture in Android.](https://github.com/qingmei2/MVVM-Rhine)

## 小结

事实上AutoDispose还有很多优秀的设计点，在源码中得以体现（包括自己实现Observable和其Observer，以及通过代理对内部的Observable进行封装，对原子操作类的灵活运用 等等）。

但无论是RxLifecycle，AutoDispose，亦或者是自己封装的util类，对于项目来说，只要满足需求，并无优劣之分，我们更希望能够从一些他人的框架中，学习到他们设计的思想，和良好的代码风格，便足矣。
 
 欢迎拍砖，一起探讨进步。
 
