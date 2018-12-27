### 本系列所有文章：

[ Android 神兵利器Dagger2使用详解（一）基础使用 ](http://www.jianshu.com/p/b40bcd1a9ec9)
[ Android 神兵利器Dagger2使用详解（二）Module&Component源码分析](http://www.jianshu.com/p/30d48ddefd30)
[ Android 神兵利器Dagger2使用详解（三）MVP架构下的使用](http://www.jianshu.com/p/c46acc3f21ab)
[ Android 神兵利器Dagger2使用详解（四）Scope注解的使用及源码分析 ](http://www.jianshu.com/p/caaac320c785)
[ 告别Dagger2模板代码：DaggerAndroid使用详解 ](http://www.jianshu.com/p/917bf39cae0d)
[ 告别Dagger2模板代码：DaggerAndroid原理解析 ](http://www.jianshu.com/p/d4d62945d9c8)
该系列首发于我的CSDN专栏 ：
[ Android开发：Dagger2详解 ](http://blog.csdn.net/column/details/17168.html)

## 概述，学Dagger2-Android的理由

### Dagger2的窘境

在使用Dagger2进行Android开发时，不可避免的问题是我们需要实例化一些Android系统的类，比如Activity或者Fragment。最理想的情况是Dagger能够创建所有需要依赖注入的对象，但事实上，我们不得不在容器的声明周期中声明这样的代码：

```
public class FrombulationActivity extends Activity {
  @Inject Frombulator frombulator;

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
}
```

这样的确可以实现Android的依赖注入，但还有两个问题需要我们去面对：

> 1.即使Dagger使我们的代码耦合性更低，但是如果要面临重构，我们仍然不得不去面对每个Activity中这样数行需要我们「复制」+「粘贴」的代码，这会给我们的重构带来一定的难度（试想一下，如果我们的应用有数十个乃至上百个这样的Activity或者Fragment容器，我们的重构计划，首先就要面对这样数百行的代码）。
> 并且随着新的开发人员加入（他们也许并不知道这些代码的意义，但是他们会复制粘贴），越来越少的人知道这些代码都干了些什么。
> 2.更重要的是，它要求注射类型（FrombulationActivity）知道其注射器。 即使这是通过接口而不是具体类型完成的，它打破了依赖注入的核心原则：一个类不应该知道如何实现依赖注入。

### 瑕不掩瑜！

诚如标题所言，虽然Dagger2在Android开发中有一些问题，但是它强大的功能依然使得开发人员趋之若鹜，无数的工程师们尝试弥补Dagger的这个问题，于是Dagger2-Android,基于Dagger2，应用于Android开发，由google开发的的拓展库应运而生。

### 本文结构

> 0. 概述，我们学Dagger2-Android的理由
> 1. 应用，Dagger2-Android入门Demo
> 2. 重构，基于google官方demo的优化尝试
> 3. 源码分析，知其然亦要知其所以然
> 4. 总结

#### 「提示」本文篇幅较长，且和Dagger2的使用有一定的区别（需要额外的学习成本），若您觉得本文尚可，建议收藏慢慢阅读

#### 「知识储备」本文可能需要您对Dagger2有足够的了解

## 应用，Dagger2-Android入门Demo

### 1.环境&依赖

```
    annotationProcessor 'com.google.dagger:dagger-compiler:2.11' //一定要添加dagger2的annotationProcessor！
    compile 'com.google.dagger:dagger-android:2.11'
    compile 'com.google.dagger:dagger-android-support:2.11' // if you use the support libraries
    annotationProcessor 'com.google.dagger:dagger-android-processor:2.11

```

### 2.Demo结构分析

首先我们先看看Demo效果

![这里写图片描述](http://upload-images.jianshu.io/upload_images/7293029-a9b34037e6ec31b4?imageMogr2/auto-orient/strip)

结构如下：

![这里写图片描述](http://upload-images.jianshu.io/upload_images/7293029-d951319c0c789186?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

可以看到，我们在MainActivity中需要通过依赖注入，获得4个对象，同时MainPresenter和MainModel也同样得到了实例化，我们先看下MVP这个架构中三个成员的代码：

### 3.MVP基本代码

#### MainActivity相关代码

MainActivity代码：

```
public class MainActivity extends BaseActivity implements MainContract.View {

    @Inject
    String className;
    @Inject
    Student student;
    @Inject
    SharedPreferences sp;
    @Inject
    MainPresenter presenter;

    @BindView(R.id.tv_content)
    TextView tvContent;

    @Override
    protected void onCreate(@Nullable Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);
        ButterKnife.bind(this);

        //下面这行代码把所有实例化的对象展示在界面中
        tvContent.setText(className + "\n" +
                student.toString() + "\n" +
                sp.toString() + "\n" +
                presenter.toString());
    }

     //省略其他代码
```

显然，从结果来看，我们根本没有在Activity中添加上文所述的模板代码：

```
    //就是这些，模板代码
    ((SomeApplicationBaseType) getContext().getApplicationContext())
        .getApplicationComponent()
        .newActivityComponentBuilder()
        .activity(this)
        .build()
        .inject(this);
```

MainPresenter代码：

```
public class MainPresenter implements MainContract.Presenter {

    private final MainActivity view;
    private final MainModel model;

    @Inject
    public MainPresenter(MainActivity view, MainModel model) {
        this.view = view;
        this.model = model;
    }

    //省略其他代码
}

```

MainModel代码：

```
public class MainModel implements MainContract.Model {

    @Inject
    public MainModel() {
    }

   //省略其他代码
}
```

和Dagger2的使用没有区别，其根本原理，都是通过@Inject注解，使Dagger2自动生成对应的工厂类，以提供MainPresenter和MainModel的实例。

#### SecondActivity代码

和MainActivity基本一样，为了简便我们只注入了一个String，用来展示类名（请参照上面的gif图）：

```
public class SecondActivity extends BaseActivity {

    @Inject
    String className;

    @Override
    protected void onCreate(@Nullable Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_second);
        TextView tv = (TextView) findViewById(R.id.tv_content);
        tv.setText(className);
    }
}
```
可以看到，我们依然没有添加冗余的模板代码，试想，一个app中这样几十个Activity容器都不需要这样的模板代码，我们的工作量确实减少了很多，同时，也达到了我们的目的，即：

> 依赖注入的核心原则：一个类不应该知道如何实现依赖注入。

实际上，我们只是在BaseActivity中，添加了这样一行代码，就实现了我们想要的效果

```
public class BaseActivity extends AppCompatActivity{

    @Override
    protected void onCreate(@Nullable Bundle savedInstanceState) {
        AndroidInjection.inject(this);  //一处声明，处处依赖注入
        super.onCreate(savedInstanceState);
    }
}

```

#### 接下来，让我们看看Dagger2-Android 的Module和Component怎么写

参考:谷歌官方Demo https://google.github.io/dagger//android.html

### 4.Module和Component

首先我们先声明对应Activity的Module和Component：

#### MainActivity依赖配置

我们声明Subcomponent,其关联的module可以提供MainActivity所需要的所有依赖的实例。
```
@Subcomponent(modules = {
        AndroidInjectionModule.class,
        MainActivitySubcomponent.SubModule.class
})
public interface MainActivitySubcomponent extends AndroidInjector<MainActivity> {

    @Subcomponent.Builder
    abstract class Builder extends AndroidInjector.Builder<MainActivity> {
    }

    @Module
    class SubModule {

        @Provides
        String provideName() {
            return MainActivity.class.getName();
        }

        @Provides
        Student provideStudent() {
            return new Student();
        }

        @Provides
        SharedPreferences provideSp(MainActivity activity) {
            return activity.getSharedPreferences("def", Context.MODE_PRIVATE);
        }

        @Provides
        MainModel provideModel() {
            return new MainModel();
        }
    }
}
```
然后我们创建MainActivityModule
```
@Module(subcomponents = MainActivitySubcomponent.class)
public abstract class MainActivityModule {

    @Binds
    @IntoMap
    @ActivityKey(MainActivity.class)
    abstract AndroidInjector.Factory<? extends Activity>
    bindMainActivityInjectorFactory(MainActivitySubcomponent.Builder builder);

}
```

ok,这样MainActivity的相关依赖注入配置完毕。

#### SecondActivity依赖配置

同理，Subcomponent代码如下：

```
@Subcomponent(modules = {
        SecondActivitySubcomponent.SubModule.class,
        AndroidInjectionModule.class,
})
public interface SecondActivitySubcomponent extends AndroidInjector<SecondActivity> {

    @Subcomponent.Builder
    abstract class Builder extends AndroidInjector.Builder<SecondActivity> {
    }

    @Module
    class SubModule {
        @Provides
        String provideName() {
            return SecondActivity.class.getName();
        }
    }
}
```
然后是Model
```
@Module(subcomponents = SecondActivitySubcomponent.class)
public abstract class SecondActivityModule {

    @Binds
    @IntoMap
    @ActivityKey(SecondActivity.class)
    abstract AndroidInjector.Factory<? extends Activity>
    bindSecondActivityInjectorFactory(SecondActivitySubcomponent.Builder builder);

}

```

> 各个模块配置完毕，接下来我们将上述的Module组合起来即可：


#### 全局依赖配置 

我们先声明我们的Application类，然后一定要记得添加到清单文件中：

```
public class MyApplication extends Application implements HasActivityInjector {

    @Inject
    DispatchingAndroidInjector<Activity> dispatchingAndroidInjector;

    @Override
    public void onCreate() {
        super.onCreate();
        //DaggerMyAppComponent.create().inject(this);
    }

    @Override
    public AndroidInjector<Activity> activityInjector() {
        return dispatchingAndroidInjector;
    }

}
```
请注意，我们在这里做了几件事：

> 1.实现HasActivityInjector接口
>
> 2.实现HasActivityInjector接口的activityInjector()方法
>
> 3.声明一个泛型为Activity的DispatchingAndroidInjector成员变量并在activityInjector()方法中返回

这里我们注释掉了一行代码，因为我们要先写AppComponent接口，然后编译，才能让编译器自动生成DaggerMyAppComponent类。

```
@Component(modules = {
        AndroidInjectionModule.class,
        AndroidSupportInjectionModule.class,
        MainActivityModule.class,
        SecondActivityModule.class
})
public interface MyAppComponent {

   void inject(MyApplication application);
}
```

请注意，我们在这里做了几件事：

> 1.在这个Component中添加了 AndroidInjectionModule 和 AndroidSupportInjectionModule
>
> 2.同时添加了MainActivityModule 和 SecondActivityModule的依赖
>
> 3.声明注入方法inject，参数类型为MyApplication

好的，所有配置全部结束，我们使用ctrl+F9（Mac:cmd+F9）进行编译,然后打开MyApplication中注释的那行代码，运行app，依赖注入成功，如上文gif图所示。

### 5.除Activity之外，Dagger-Android还提供了Fragment，BroadCast,Service等其他组件的注入方式，详情请参照官方网站，本文不赘述。

https://google.github.io/dagger//android.html

### 6.注意

如果您按照本文中的代码实现，并没有得到对应的效果或者编译错误，请仔细检查是否严格按照本文的步骤进行编码，如果还有问题，建议您直接拉到本文最下方，下载源码查看。


## 重构，基于google官方demo的优化尝试

### 1.还有一些冗余代码

上述章节中的使用方式基本是按照谷歌官方的guide进行的依赖注入，实际上个人认为还有几点可以优化：

*1.多余的ActivityModule*

请参照上文的MainActivityModule和SecondActivityModule,实际上从根本上来说，也类似模板代码，当界面多了，会生成大量的java文件

*2.多余的Subcomponent*
同理，太多了。

*3.越来越庞大的AppComponent* 
当Activity越来越多，AppComponet中的注解会变成这样：

```
@Component(modules = {
        AndroidInjectionModule.class,
        AndroidSupportInjectionModule.class,
        MainActivityModule.class,
        SecondActivityModule.class
        ...
        ...//中间省略不知道多少行
        XXActivityModule.class
})
```

我们来看看目前的结构：
![这里写图片描述](http://upload-images.jianshu.io/upload_images/7293029-9c863dee3256ce55?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

可以看到，很冗余，实质上，表面上的注入器（Appcomponent）,和对象的提供者（MainSubComponent.SubModule）两者起到了至关重要的作用，而Module和SubComponent作用不大。

### 2.假设

于是我们联想，我们可不可以改变结构成这样呢：

![这里写图片描述](http://upload-images.jianshu.io/upload_images/7293029-6dd080b466e8ff43?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

简述如下：

>1.AppComponent的module:AllActivityModule，其负责所有ActivityModule实例的管理
>
>2.BaseActivityComponent为AllActivityModule的subcomponent,管理了提供各activity所需对象的module
>
>3.MainActivityModule 负责提供所有对应Activity所需要对象实例的提供。

### 3.实装

#### 首先，创建BaseActivityComponent，并将所有的ActivitySubcomponent删除：

```
@Subcomponent(modules = {
        AndroidInjectionModule.class,
        MainActivityModule.class,   //管理MainActivity所需对象的提供
        SecondActivityModule.class  //管理SecondActivity所需对象的提供
})
public interface BaseActivityComponent extends AndroidInjector<BaseActivity> {

    @Subcomponent.Builder
    abstract class Builder extends AndroidInjector.Builder<BaseActivity> {
    }

}
```

#### 然后我们将对应的对象提供放到ActivityModule里：

MainActivityModule:

```
@Module
public class MainActivityModule {

    @Provides
    static String provideName() {
        return MainActivity.class.getName();
    }

    @Provides
    static SharedPreferences provideSp(MainActivity activity) {
        return activity.getSharedPreferences("def", Context.MODE_PRIVATE);
    }
    //其他对象的提供 省略
}
```

SecondActivityModule:

```
@Module
public abstract class SecondActivityModule {

    @Provides
    static String provideName() {
        return SecondActivity.class.getName();
    }

}
```


那么问题来了，我们原本放在这里面的这些代码怎么办呢，例如：
```
    @Binds
    @IntoMap
    @ActivityKey(SecondActivity.class)
    abstract AndroidInjector.Factory<? extends Activity>
    bindSecondActivityInjectorFactory(SecondActivitySubcomponent.Builder builder);
```

#### 新建AllActivitysModule，并统统放入其中统一管理：

```
@Module(subcomponents = {
        BaseActivityComponent.class
})
public abstract class AllActivitysModule {

    @ContributesAndroidInjector(modules = MainActivityModule.class)
    abstract MainActivity contributeMainActivitytInjector();

    @ContributesAndroidInjector(modules = SecondActivityModule.class)
    abstract SecondActivity contributeSecondActivityInjector();

}
```

这样，我们每次新建一个Activity，只需要统一在这里新建2行代码声明对应的Activity即可。

关于这两行代码的解释，请参考官网：https://google.github.io/dagger//android.html

>Pro-tip: If your subcomponent and its builder have no other methods or supertypes than the ones mentioned in step #2, you can use @ContributesAndroidInjector to generate them for you. Instead of steps 2 and 3, add an abstract module method that returns your activity, annotate it with @ContributesAndroidInjector, and specify the modules you want to install into the subcomponent. If the subcomponent needs scopes, apply the scope annotations to the method as well.

>@ActivityScope
@ContributesAndroidInjector(modules = { /* modules to install into the subcomponent */ })
abstract YourActivity contributeYourActivityInjector();


#### 最后修改AppComponent

```
@Component(modules = {
        AndroidInjectionModule.class,
        AndroidSupportInjectionModule.class,
        AllActivitysModule.class    
})
public interface MyAppComponent {

    void inject(MyApplication application);
}
```

#### 修改完毕，大功告成！
好的，所有配置全部结束，我们使用ctrl+F9（Mac:cmd+F9）进行编译,然后打开MyApplication中注释的那行代码，运行app，依赖注入依旧成功。

不同的是，这次我们的模板代码更少了。

## 源码分析，知其然亦要知其所以然

一路写下来，单单该库的使用，竟然洋洋洒洒竟然写了这么多，实在超出预料。

// 源码分析，鉴于篇幅所限，准备另起一篇文章改日慢慢分析。 2017/8/22

### 2017/8/30更新
源码分析，鉴于篇幅所限，另起一篇文章，请阅读本文：

[ 告别Dagger2模板代码：DaggerAndroid原理解析 ](http://www.jianshu.com/p/d4d62945d9c8)

## 总结

看了这么多，有的朋友可能会看了几眼就不看了，也一定会有人觉得笔者啰嗦，文笔有限，但还是希望能够给大家提供一定的帮助，毕竟关于这个库，国内的中文相关资料实在有限，而Dagger依赖注入库的学习成本，也是出了名的高。


最后提供一下本文demo源码连接：

https://github.com/qingmei2/Sample_dagger2
