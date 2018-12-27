![](http://upload-images.jianshu.io/upload_images/7293029-aa9d6e31210b87a8.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
##前言

### 本系列所有文章：

[ Android 神兵利器Dagger2使用详解（一）基础使用 ](http://www.jianshu.com/p/b40bcd1a9ec9)
[ Android 神兵利器Dagger2使用详解（二）Module&Component源码分析](http://www.jianshu.com/p/30d48ddefd30)
[ Android 神兵利器Dagger2使用详解（三）MVP架构下的使用](http://www.jianshu.com/p/c46acc3f21ab)
[ Android 神兵利器Dagger2使用详解（四）Scope注解的使用及源码分析 ](http://www.jianshu.com/p/caaac320c785)
[ 告别Dagger2模板代码：DaggerAndroid使用详解 ](http://www.jianshu.com/p/917bf39cae0d)
[ 告别Dagger2模板代码：DaggerAndroid原理解析 ](http://www.jianshu.com/p/d4d62945d9c8)
该系列首发于我的CSDN专栏 ：
[ Android开发：Dagger2详解 ](http://blog.csdn.net/column/details/17168.html)

在我的上一篇文章中，我们通过一点点分析@Module、@Inject以及@Component注解生成的源码，了解了Dagger2依赖注入魔法的根源：

>1、 **@Inject** 注解构造 生成“大众”工厂类 或者 **@Module +@Providers** 提供注入“私有”工厂类
>
>2、通过Component 创建获得**Activity**,获得工厂类**Provider**，统一交给**Injector**
>
>3、Injector将Provider的get()方法提供的对象，注入到Activity容器对应的成员变量中
>
>我们就可以直接使用容器（Activity）中对应的成员变量了

借用一张图更形象一些：[（图片来源地址）感谢@牛晓伟](http://www.jianshu.com/p/1d42d2e6f4a5)

![这里写图片描述](http://upload-images.jianshu.io/upload_images/1504173-0b81f8a57768a703.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

但是上文的使用方法实在有些简陋，不足以体现Dagger2为什么好用，或者说好用在哪，本文将会以正常实际开发为例，阐述Dagger2实际开发中的使用方式。

##一、全局类 AppComponent & AppModule

我们先简单将我们开发时需要依赖注入的对象进行简单的分类：

> **1、全局变量**
> 这类变量一般为单例模式存在，比如**SharedPerferences对象、Application对象，还有ServiceManager（网络请求API管理类）**等等，这些东西也许我们只想初始化一次，之后处处使用就好了。
> **2、局部变量**
> 这些变量可能我们只想在某个地方使用，比如Student对象，我们一般也不会只声明一次。

开发中，局部变量尚还好说，我们在没有Dagger的时候，用的时候new一个就可以了，类似于SharedPerferences这种对象我们就比较头疼，也没有必要每次都去context.getSharedPerferences()吧，但是我们有了Dagger，我们可以通过依赖注入的方式，仅仅初始化一次，之后每次想使用，只需要@Inject注解，就能直接使用了，是不是很方便呀。

说干就干，我们创建一个全局的AppComponent 和 AppModule：
#### 0、首先创建一个自定义的Application类：

```
public class MyApplication extends Application{
    @Override
    public void onCreate() {
        super.onCreate();
    }
}

```
**记得在清单文件中使用这个类哦 ~**
####1、创建AppModule，提供你想要提供的全局变量

```
@Module
public class AppModule {

    private MyApplication application;

    public AppModule(MyApplication application) {
        this.application = application;
    }
    
	//提供全局的sp对象
    @Provides
    SharedPreferences provideSharedPreferences(){
        return application.getSharedPreferences("spfile", Context.MODE_PRIVATE);
    }
    
	//提供全局的Application对象
    @Provides
    MyApplication provideApplication(){
        return application;
    }
	
	//你还可以提供更多.......
}

```

####2、创建AppComponent，然后ctrl+F9编译

```
@Component(modules = AppModule.class)
public interface AppComponent {

    SharedPreferences sharedPreferences();

    MyApplication myApplication();
}

```

####3、编译成功后，修改Application类，把Component注入器注入你的Application中：

```
public class MyApplication extends Application{

    @Override
    public void onCreate() {
        super.onCreate();
        inject();
    }

    private void inject() {
        AppComponent appComponent = DaggerAppComponent.builder()
                .appModule(new AppModule(this))
                .build();
        ComponentHolder.setAppComponent(appComponent);
    }
}
```
ComponentHolder类，方便获取appComponent对象：

```
public class ComponentHolder {
    private static AppComponent myAppComponent;

    public static void setAppComponent(AppComponent component) {
        myAppComponent = component;
    }

    public static AppComponent getAppComponent() {
        return myAppComponent;
    }
}

```
ok，这样，无论何处，我们需要获得AppComponent对象时，只需要调用ComponentHolder.getAppComponent()就可以啦！

####**注意：**

> **我们需要知道刚才这些文件的作用：**

>**1、AppModule**：初始化全局变量
>**2、AppComponent：**注入器，储存了我们要用到的全局变量对象
>**3、MyApplication:** 在onCreate()方法中唯一一次初始化了AppComponent对象，并放入了ComponentHolder中。

>**MyApplication （作为参数初始化）-> AppModule(初始化全局变量) -> (注入) AppComponent ->(存储到)ComponentHolder**

好的，现在我们已经把所有要用到的全局变量存到了AppComponent中，接下来我们创建一个普通的容器Activity。

##二、日常开发 Activity相关

####**0、我们先创建一个普通的Module和Component**

```
@Module
public class A02Module {

    private A02Activity activity;

    public A02Module(A02Activity activity) {
        this.activity = activity;
    }
	
	//显然我们并不是很多地方都需要某对象，我们在需要用该对象的界面的Module中提供注入即可
    @Provides
    Student provideStudent() {
        return new Student();
    }

}
```

```
@Component(modules = A02Module.class, dependencies = AppComponent.class)
public interface A02Component {

    void inject(A02Activity activity);

}

```
请注意Component接口，我们发现多了这样一行代码：

> **dependencies = AppComponent.class**

其实也很好理解，比如说我们需要Student对象和SharedPerferences对象，前者是A02Component提供的，后者则需要「依赖（dependencies）」AppComponent提供，等于这次inject(A02Activity activity)依赖注入需要两个Component共同出力完成。

####**1、编译成功后，在Activity中声明：**

```
public class A02Activity extends AppCompatActivity {

    @Inject
    Student student;
    @Inject
    SharedPreferences sp;

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_a02);
        inject();
    }

    private void inject() {
        DaggerA02Component.builder()
                .appComponent(ComponentHolder.getAppComponent())//添加appComponent注入器
                .a02Module(new A02Module(this))
                .build()
                .inject(this);

        Log.i("tag", "注入完毕，Student = " + student.toString());
        Log.i("tag", "注入完毕，sp = " + sp.toString());
    }
}

```

查看一下结果，依赖注入成功：

![这里写图片描述](http://upload-images.jianshu.io/upload_images/7293029-dab2db7209dc8a40?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

是不是很神奇！试想一下，如果我们有很多类似的对象需要经常使用，我们再也不需要每次都去初始化，直接@Inject，然后直接使用就可以了;

更方便的是，如果我们需要修改（比如SharedPreferences文件的名字等等），只需要跑到提供它的Module文件中：

```
@Module
public class AppModule {

	...
	...
	...
	
	//提供全局的sp对象 -> 修改
    @Provides
    SharedPreferences provideSharedPreferences(){
     //   return application.getSharedPreferences("spfile", Context.MODE_PRIVATE);
        return application.getSharedPreferences("New spfile", Context.MODE_PRIVATE);
    }

	...
	...
	...
}

```
当然，你的Activity容器中代码也能变得更加赏心悦目：

![这里写图片描述](http://upload-images.jianshu.io/upload_images/7293029-c170a367428addb5?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

##三*、源码分析：
因为在我的上一篇文章[Android 神兵利器Dagger2使用详解（二）Module&Component源码分析](http://blog.csdn.net/mq2553299/article/details/73136396) 中已经对原理有了基本的了解，所以我们可以轻松看懂实现原理。

简单分析下源码，如下为编译器自动生成的代码：

![这里写图片描述](http://upload-images.jianshu.io/upload_images/7293029-4aa09457610baf05?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

####**1、App全局类源码（global包下的三个类）**

> **提供对象的工厂类：**
> AppModule_ProvideApplicationFactory.java
> AppModule_ProvideSharedPreferencesFactory.java
> **注入器：**
> DaggerAppComponent.java

工厂类直接略过，只不过是通过传入Module对象初始化，然后调用get（）方法获得Module中对应的对象罢了（若不清楚请参考我的[上一篇文章](http://blog.csdn.net/mq2553299/article/details/73136396)，分析得比较清楚）。

注入器的改变：

```
public final class DaggerAppComponent implements AppComponent {

  private Provider<SharedPreferences> provideSharedPreferencesProvider;

  private Provider<MyApplication> provideApplicationProvider;

  private DaggerAppComponent(Builder builder) {
	...
    initialize(builder);
  }

	...
  @SuppressWarnings("unchecked")
  private void initialize(final Builder builder) {
	//2.初始化2个工厂类
    this.provideSharedPreferencesProvider =
        AppModule_ProvideSharedPreferencesFactory.create(builder.appModule);

    this.provideApplicationProvider = AppModule_ProvideApplicationFactory.create(builder.appModule);
  }
	
	//3.想要sp对象吗？appComponent调用一下这个方法就有了
  @Override
  public SharedPreferences sharedPreferences() {
    return provideSharedPreferencesProvider.get();
  }

  @Override
  public MyApplication myApplication() {
    return provideApplicationProvider.get();
  }

  public static final class Builder {
	  //1、参数传入，初始化appModule
	  private AppModule appModule;
	   ...
	   ...
  }
}
```
还是老三样，三板斧，没什么难度。

####**2、A02Component源码分析：**
> **提供Student对象的工厂类：**
> A02Module_ProvideStudentFactory.java
> **把对象注入到容器的Injcetor类：**
> A02Activity_MembersInjector.java
> **注入器：**
> DaggerA02Component.java

1、工厂略过不提，提供Student对象的类而已；
2、Injector 我们上一篇文章中可以知道，通过Component把需要对象的容器(activity)、以及对象工厂（studentProvider）给Injector后，Injector把对象赋值给容器中对应的属性而已，比如：

> activity.student = studentProvider.get();

3、DaggerA02Component，**执行顺序参考代码中注释**:

```
public final class DaggerA02Component implements A02Component {
  private Provider<Student> provideStudentProvider;

  private Provider<SharedPreferences> sharedPreferencesProvider;

  private MembersInjector<A02Activity> a02ActivityMembersInjector;

  private DaggerA02Component(Builder builder) {
    assert builder != null;
    initialize(builder);
  }

  public static Builder builder() {
    return new Builder();
  }
	
  @SuppressWarnings("unchecked")
  private void initialize(final Builder builder) {
  
	//3.初始化所有对象的工厂Provider
	//3.1 从builder中的Module中获得学生Student工厂
    this.provideStudentProvider = A02Module_ProvideStudentFactory.create(builder.a02Module);
    
     //3.2 从builder中的appComponent中获得SP工厂
    this.sharedPreferencesProvider =
        new Factory<SharedPreferences>() {
          private final AppComponent appComponent = builder.appComponent;

          @Override
          public SharedPreferences get() {
            return Preconditions.checkNotNull(
                appComponent.sharedPreferences(),
                "Cannot return null from a non-@Nullable component method");
          }
        };
	//4、把所有工厂都传到Injector中
    this.a02ActivityMembersInjector =
        A02Activity_MembersInjector.create(provideStudentProvider, sharedPreferencesProvider);
  }

  @Override
  public void inject(A02Activity activity) {
	 //5、把activity放入到Injector中（接下来结果很明显了...）
    a02ActivityMembersInjector.injectMembers(activity);
  }

  public static final class Builder {
    private A02Module a02Module;

    private AppComponent appComponent;

    private Builder() {}

    public A02Component build() {
      if (a02Module == null) {
        throw new IllegalStateException(A02Module.class.getCanonicalName() + " must be set");
      }
      if (appComponent == null) {
        throw new IllegalStateException(AppComponent.class.getCanonicalName() + " must be set");
      }
      return new DaggerA02Component(this);
    }
	//2.传入a02Module
    public Builder a02Module(A02Module a02Module) {
      this.a02Module = Preconditions.checkNotNull(a02Module);
      return this;
    }
	//1.传入appComponent
    public Builder appComponent(AppComponent appComponent) {
      this.appComponent = Preconditions.checkNotNull(appComponent);
      return this;
    }
  }
}
```
代码虽然变多了，但是，还是很好理解...

##小结

不知道大家有没有感到Dagger2的好用，确实是开发时解耦神器啊！

膜拜！

如果有地方看不懂，建议自己尝试敲两遍，然后参考编译器生成的代码，多想想，可能一开始会有点绕，但还是可以看懂的。

[代码已托管Github，源码传送门，点我查看](https://github.com/qingmei2/Sample_dagger2)

在我的下一篇[ Android 神兵利器Dagger2使用详解（四）Scope注解的使用及源码分析 ](http://www.jianshu.com/p/caaac320c785)中，将会结合源码，学习并加深**@Scope注解**。

## 2017/8/24日更新

 事实上，Android开发中使用Dagger,开发人员仍然需要面对一些问题。

google工程师们尝试弥补Dagger的问题，于是Dagger2-Android,基于Dagger2，应用于Android开发，由google开发的的拓展库应运而生：

* [ 告别Dagger2模板代码：DaggerAndroid使用详解 ](http://www.jianshu.com/p/917bf39cae0d)
* [ 告别Dagger2模板代码：DaggerAndroid原理解析 ](http://www.jianshu.com/p/d4d62945d9c8)
