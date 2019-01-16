### 本系列所有文章：

[ Android 神兵利器Dagger2使用详解（一）基础使用 ](http://www.jianshu.com/p/b40bcd1a9ec9)
[ Android 神兵利器Dagger2使用详解（二）Module&Component源码分析](http://www.jianshu.com/p/30d48ddefd30)
[ Android 神兵利器Dagger2使用详解（三）MVP架构下的使用](http://www.jianshu.com/p/c46acc3f21ab)
[ Android 神兵利器Dagger2使用详解（四）Scope注解的使用及源码分析 ](http://www.jianshu.com/p/caaac320c785)
[ 告别Dagger2模板代码：DaggerAndroid使用详解 ](http://www.jianshu.com/p/917bf39cae0d)
[ 告别Dagger2模板代码：DaggerAndroid原理解析 ](http://www.jianshu.com/p/d4d62945d9c8)
该系列首发于我的CSDN专栏 ：
[ Android开发：Dagger2详解 ](http://blog.csdn.net/column/details/17168.html)

## 概述

距离我的上一篇文章发布以来，有幸收获了一些朋友的认可，我很开心。

在上一篇文章中，我简单叙述了Dagger2这个库目前在Android开发中的一些短板，为什么我们学习Dagger-Android这个拓展库，以及如何使用这个拓展库。

今天我们进行代码分析，看看Dagger-Android是如何基于Dagger2实现一行代码实现所有同类组件依赖注入的。

### 阅读本文也许您需要准备的

* 本文可能需要您对Dagger2有一定的了解，包括它内部模板代码生成的一些逻辑。

## 核心代码 

书承上文，我们知道，我们实现依赖注入的代码主要为以下两行：

```java
public class MyApplication extends Application implements HasActivityInjector {

    @Inject
    DispatchingAndroidInjector<Activity> dispatchingAndroidInjector;

    @Override
    public void onCreate() {
        super.onCreate();
        //1.Application级的依赖注入
        DaggerMyAppComponent.create().inject(this);
    }

    @Override
    public AndroidInjector<Activity> activityInjector() {
        return dispatchingAndroidInjector;
    }

}
```


```java
public class BaseActivity extends AppCompatActivity{

    @Override
    protected void onCreate(@Nullable Bundle savedInstanceState) {
        //2.Activity级的依赖注入
        AndroidInjection.inject(this);
        super.onCreate(savedInstanceState);
    }
}
```

抛开这两行，也许我们还有更多疑问：

* 1.MyApplication中的dispatchingAndroidInjector成员是干嘛的
* 2.MyApplication实现的HasActivityInjector接口和实现的activityInjector()方法又是干嘛的
* 3.为什么这样2行依赖注入代码，就能代替之前我们每个Activity都需要写的模板代码呢：

```java
  @Override
  public void onCreate(Bundle savedInstanceState) {
    super.onCreate(savedInstanceState);
    // DO THIS FIRST. Otherwise frombulator might be null!
    ((SomeApplicationBaseType) getContext().getApplicationContext())
        .getApplicationComponent()
        .newActivityComponentBuilder()
        .activity(this)
        .build()
        .inject(this);
    // ... now you can write the exciting code
  }
```

直接公布结论：
  
![这里写图片描述](http://upload-images.jianshu.io/upload_images/7293029-13da860fc0dd97bb?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240) 


![这里写图片描述](http://upload-images.jianshu.io/upload_images/7293029-52968e8570b2c292?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

  ![](http://upload-images.jianshu.io/upload_images/7293029-e738e57c52650949?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
  
  这三张图描述了很多东西，我们不需要先去看懂它，我们会随着我们的步骤，一步步分析透彻它，拨开云雾见晴天。、
  
  首先我们分析Application中的这行代码：
  
>  DaggerMyAppComponent.create().inject(this);  
  
## 一、DaggerMyAppComponent.create()原理

我们先点开create()方法，可以看到，无非是初始化了一个DaggerMyAppComponent对象，然后执行了initialize()方法
```java
    public static MyAppComponent create() {
        return new Builder().build();
    }
    
    public static final class Builder {
        private Builder() {
        }

        public MyAppComponent build() {
            return new DaggerMyAppComponent(this);
        }
    }
        
    private DaggerMyAppComponent(Builder builder) {
        assert builder != null;
        initialize(builder);
    }
```

我们接着看initialize()方法：
```
private void initialize(final Builder builder) {
    //1
    this.mainActivitySubcomponentBuilderProvider =
            new dagger.internal.Factory<
                    AllActivitysModule_ContributeMainActivitytInjector.MainActivitySubcomponent.Builder>() {
                @Override
                public AllActivitysModule_ContributeMainActivitytInjector.MainActivitySubcomponent.Builder
                get() {
                    return new MainActivitySubcomponentBuilder();
                }
            };

    this.bindAndroidInjectorFactoryProvider = (Provider) mainActivitySubcomponentBuilderProvider;

    this.secondActivitySubcomponentBuilderProvider =
            new dagger.internal.Factory<
                    AllActivitysModule_ContributeSecondActivityInjector.SecondActivitySubcomponent
                            .Builder>() {
                @Override
                public AllActivitysModule_ContributeSecondActivityInjector.SecondActivitySubcomponent
                        .Builder
                get() {
                    return new SecondActivitySubcomponentBuilder();
                }
            };

    this.bindAndroidInjectorFactoryProvider2 = (Provider) secondActivitySubcomponentBuilderProvider;
    //2
    this.mapOfClassOfAndProviderOfFactoryOfProvider =
            MapProviderFactory
                    .<Class<? extends Activity>, AndroidInjector.Factory<? extends Activity>>builder(2)
                    .put(MainActivity.class, bindAndroidInjectorFactoryProvider)
                    .put(SecondActivity.class, bindAndroidInjectorFactoryProvider2)
                    .build();
    //3
    this.dispatchingAndroidInjectorProvider =
            DispatchingAndroidInjector_Factory.create(mapOfClassOfAndProviderOfFactoryOfProvider);
    //4
    this.myApplicationMembersInjector =
            MyApplication_MembersInjector.create(dispatchingAndroidInjectorProvider);
}
```
这就有点难受了，代码很多，我们一步步慢慢分析。

我们不要一次想着把这些代码彻底读懂，我们首先大概对整个流程有个了解，之后慢慢拆分代码不迟：

 * 1. 我们可以理解为直接初始化了MainActivitySubcomponentBuilder和SecondActivitySubcomponentBuilder的对象，这个对象能够提供一个对应ActivityComponent的实例化对象，有了这个实例化对象，我们就能够对对应的Activity进行依赖注入。
 2. 这个很明显了，把所有ActivityComponent的实例化对象都存储到一个Map中，然后这个Map通过放入mapOfClassOfAndProviderOfFactoryOfProvider对象中。
3. 通过mapOfClassOfAndProviderOfFactoryOfProvider初始化dispatchingAndroidInjectorProvider对象
4. 通过dispatchingAndroidInjectorProvider对象初始化myApplicationMembersInjector对象

好的基本差不多了 我们看看每个步骤都是如何实现的

### 第1步详细分析
以SecondActivity的这行代码为例：
```
this.secondActivitySubcomponentBuilderProvider =
            new dagger.internal.Factory<    //1.看这行，实际上是new了一个能提供SecondActivitySubcomponentBuilder()的Provider
                    AllActivitysModule_ContributeSecondActivityInjector.SecondActivitySubcomponent
                            .Builder>() {
                @Override
                public AllActivitysModule_ContributeSecondActivityInjector.SecondActivitySubcomponent
                        .Builder
                get() {
                //2.这个SecondActivitySubcomponentBuilder能干什么呢
                    return new SecondActivitySubcomponentBuilder();
                }
            };
```

```
 private final class SecondActivitySubcomponentImpl
            implements AllActivitysModule_ContributeSecondActivityInjector.SecondActivitySubcomponent {
        private MembersInjector<SecondActivity> secondActivityMembersInjector;
        //3.我们点开构造方法，看到实际是走了initialize(builder);
        private SecondActivitySubcomponentImpl(SecondActivitySubcomponentBuilder builder) {
            assert builder != null;
            initialize(builder);
        }

        @SuppressWarnings("unchecked")
        private void initialize(final SecondActivitySubcomponentBuilder builder) {
            //4.实际上是初始化了一个SecondActivity_MembersInjector，这个Injector能够对SecondActivity所需要的对象提供注入
            this.secondActivityMembersInjector =
                    SecondActivity_MembersInjector.create(SecondActivityModule_ProvideNameFactory.create());
        }

        @Override
        public void inject(SecondActivity arg0) {
            //5.我们不难想象，不久后当SecondActivity需要依赖注入，一定会执行这行代码进行对象注入
            secondActivityMembersInjector.injectMembers(arg0);
        }
}
```

看到这里我们明白了，实际上第一步是ApplicationComponent实例化了所有的ActivityComponent（就是那个能提供SecondActivitySubcomponentBuilder()的Provider）

然后每个ActivityComponent都能提供某个Activity依赖注入所需要的ActivityMembersInjector。

### 第2步第3步分析
第2步就很简单了，这些ActivityComponent都被放入了一个Map集合中，该集合又放入了apOfClassOfAndProviderOfFactoryOfProvider中，这个Provider实际上也是一个工厂，负责提供该Map的实例化

第3步也很好理解，我们MapProvider工厂交给了就是dispatchingAndroidInjectorProvider去管理。

### 第4步分析：

实际上我们传入dispatchingAndroidInjectorProvider做了什么呢？

我们看看MyApplication_MembersInjector对象是干嘛的:

```
//核心代码
public final class MyApplication_MembersInjector implements MembersInjector<MyApplication> {
  private final Provider<DispatchingAndroidInjector<Activity>> dispatchingAndroidInjectorProvider;
  ...
  ...
  ...
  //依赖注入Application!
  @Override
  public void injectMembers(MyApplication instance) {
    instance.dispatchingAndroidInjector = dispatchingAndroidInjectorProvider.get();
  }
}
```
大家可以明白了，原来第四步初始化该对象时，会将传入的dispatchingAndroidInjectorProvider存储起来，将来用到的时候将其对象注入到需要注入的Application中:

```java
public class MyApplication extends Application implements HasActivityInjector {
    //就是注入给这个成员变量！
    @Inject
    DispatchingAndroidInjector<Activity> dispatchingAndroidInjector;
    //省略其他代码
    ...
    ...
}
```

### 整理思路

现在我们再来看一下这张图，相信您已经对这张图有所领会了。

![这里写图片描述](http://upload-images.jianshu.io/upload_images/7293029-13da860fc0dd97bb?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240) 


可以看到，Application进行初始化时，仅仅一部DaggerMyAppComponent.create()，就已经处理了如此多的事件，初始化了这么多工厂类，等待我们去调用他们实现依赖注入了。
  
## 二、DaggerMyAppComponent.create().inject(this)原理

其实这个步骤就是确定一下我们上面的对Dagger-Android这个库的分析是否是对的，我们来看这行代码

```
public final class DaggerMyAppComponent implements MyAppComponent {
    ...
    ...
    ...
    
    @Override
    public void inject(MyApplication application) {
        myApplicationMembersInjector.injectMembers(application);
    }
}
```
这就和我们上面第四步的分析吻合了:

```
//我们刚才分析过的，再看一遍
//核心代码
public final class MyApplication_MembersInjector implements MembersInjector<MyApplication> {
  private final Provider<DispatchingAndroidInjector<Activity>> dispatchingAndroidInjectorProvider;
  ...
  ...
  ...
  //依赖注入Application!
  @Override
  public void injectMembers(MyApplication instance) {
    instance.dispatchingAndroidInjector = dispatchingAndroidInjectorProvider.get();
  }
}

public class MyApplication extends Application implements HasActivityInjector {
    //就是注入给这个成员变量！
    @Inject
    DispatchingAndroidInjector<Activity> dispatchingAndroidInjector;
    //省略其他代码
    ...
    ...
}
```

### 整理思路

![这里写图片描述](http://upload-images.jianshu.io/upload_images/7293029-52968e8570b2c292?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


可以看到，这一步实际上我们做了至关重要的一步，就是把Application中的这个不知道是干什么的成员变量进行了初始化！！！

>    @Inject
     DispatchingAndroidInjector<Activity> dispatchingAndroidInjector;
     
我们现在知道，这个dispatchingAndroidInjector中，实际拥有一个Map,Map中有着所有Activity所需依赖注入使用的Component!

可以说，这个看起来平凡无奇的成员变量，才是我们整个项目依赖注入的大管家！

### 顿悟

相信如果您认真一步步看到这里，对于下面的2个问题，您已经有了自己的答案：

* 1.MyApplication中的dispatchingAndroidInjector成员是干嘛的

## 三、 AndroidInjection.inject(this)原理

我们在初始化一个Activity，必然会调用BaseActivity中的这行代码：

>AndroidInjection.inject(this)

我们好奇，为什么仅仅在BaseActivity这样1行依赖注入代码，就能代替之前我们每个Activity都需要写的模板代码呢？

答案近在眼前。

```
public final class AndroidInjection {
  ...
  ...
  ...
  public static void inject(Activity activity) {
    //核心代码如下
    //1.获取Application
    Application application = activity.getApplication();
    
    //2.通过activityInjector（）获取dispatchingAndroidInjector对象！
    AndroidInjector<Activity> activityInjector =
        ((HasActivityInjector) application).activityInjector();
        
    //3.dispatchingAndroidInjector依赖注入
    activityInjector.inject(activity);
  }
}
```

前两步毫无技术含量，我们继续看activityInjector.inject(activity)：

我们需要找到真正的activityInjector对象，就是DispatchingAndroidInjector：

```
public final class DispatchingAndroidInjector<T> implements AndroidInjector<T> {

  @Override
  public void inject(T instance) {  //instance在这里为Activity
    //1.执行maybeInject()方法
    boolean wasInjected = maybeInject(instance);
  }
  
  public boolean maybeInject(T instance) {  //instance在这里为Activity
    //2.获取对应Provider
    Provider<AndroidInjector.Factory<? extends T>> factoryProvider =
        injectorFactories.get(instance.getClass());
    
    AndroidInjector.Factory<T> factory = (AndroidInjector.Factory<T>) factoryProvider.get();
     
     //3.获取对应的ActivityMembersInjector
     AndroidInjector<T> injector =
              factory.getClass().getCanonicalName());
      //4.注入到对应的Activity中
      injector.inject(instance);
  }
}
```

### 整理思路

  ![](http://upload-images.jianshu.io/upload_images/7293029-e738e57c52650949?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

可以看到，这一步实际上我们只是取得Application中的大管家dispatchingAndroidInjector，通过它取得了对应Activity的Inject实例，并进行依赖注入。

### 顿悟

相信如果您认真一步步看到这里，对于下面的2个问题，您已经有了自己的答案：

* 2.MyApplication实现的HasActivityInjector接口和实现的activityInjector()方法又是干嘛的
* 3.为什么这样2行依赖注入代码，就能代替之前我们每个Activity都需要写的模板代码呢：


### 至此，AndroidInject原理分析基本也到了尾声

## 附录

最后提供一下本文demo源码连接和图片资源，都已放到Github：

https://github.com/qingmei2/Sample_dagger2

[本文所使用的高清图片链接](https://github.com/qingmei2/Sample_dagger2/blob/master/DaggerAndroid%E5%8E%9F%E7%90%86%E5%88%86%E6%9E%90.png)
