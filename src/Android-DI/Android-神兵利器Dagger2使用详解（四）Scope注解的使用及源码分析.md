![](http://upload-images.jianshu.io/upload_images/7293029-6d31a84251aa3be9.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

## 前言
### 本系列所有文章：

[ Android 神兵利器Dagger2使用详解（一）基础使用 ](http://www.jianshu.com/p/b40bcd1a9ec9)
[ Android 神兵利器Dagger2使用详解（二）Module&Component源码分析](http://www.jianshu.com/p/30d48ddefd30)
[ Android 神兵利器Dagger2使用详解（三）MVP架构下的使用](http://www.jianshu.com/p/c46acc3f21ab)
[ Android 神兵利器Dagger2使用详解（四）Scope注解的使用及源码分析 ](http://www.jianshu.com/p/caaac320c785)
[ 告别Dagger2模板代码：DaggerAndroid使用详解 ](http://www.jianshu.com/p/917bf39cae0d)
[ 告别Dagger2模板代码：DaggerAndroid原理解析 ](http://www.jianshu.com/p/d4d62945d9c8)
该系列首发于我的CSDN专栏 ：
[ Android开发：Dagger2详解 ](http://blog.csdn.net/column/details/17168.html)

在我的上一篇文章中，我们以简单的案例对Dagger2依赖注入库在实际开发中的使用方法进行了学习。

本文内容：

> 1.@Singleton **全局单例** 注解的使用
> 
> 2.自定义@Scope **局部单例** 注解的使用
> 
>3.通过 **源码分析 **@Singleton和@Scope注解是 **如何实现单例** 的。


## 一.@Singleton:全局单例注解

承接上文，我们在AppModule中提供了以下对象的初始化：

```
@Module
public class AppModule {

    private MyApplication application;

    public AppModule(MyApplication application) {
        this.application = application;
    }

    //提供sp对象
    @Provides
    SharedPreferences provideSharedPreferences(){
        return application.getSharedPreferences("spfile", Context.MODE_PRIVATE);
    }

    //提供Application对象
    @Provides
    MyApplication provideApplication(){
        return application;
    }
}
```
我们肯定会有这样一种需求，类似于SharedPreferences对象，我们可能在整个App的生命周期中只需要一个单例，而不需要每次都通过application.getSharedPreferences("spfile", Context.MODE_PRIVATE)获得新的对象，那么怎么办呢，我们只需要在你需要单例的对象提供方法上加一个注解@Singleton ,类似这样：

```
@Module
public class AppModule {

    private MyApplication application;

    public AppModule(MyApplication application) {
        this.application = application;
    }
	
	//全局单例SharedPreferences
    @Provides
    @Singleton
    SharedPreferences provideSharedPreferences() {
        return application.getSharedPreferences("spfile", Context.MODE_PRIVATE);
    }

    @Provides
    MyApplication provideApplication() {
        return application;
    }
 }
```
只需要一个@Singleton,无论我们在多少个Activity容器中通过@Inject获取这个SharedPreferences对象的实例，该对象都是同一个对象，从而实现了全局单例的效果。

非常简单的使用方式，**你需要什么对象全局单例，就在提供该对象方法的@Provides注解旁加上一个@Singleton,并且在该module关联的Component中加上同样的注解**：

```
@Singleton		//不要忘了还要在关联的Component接口中声明，否则会编译报错
@Component(modules = AppModule.class)
public interface AppComponent {

    SharedPreferences sharedPreferences();

    MyApplication myApplication();

}
```

ok，使用方式很简单。

##  二.自定义@Scope 局部单例注解

事实上，除了全局单例，我们可能在开发中更多的是需要**局部单例**，比如，我在ActivityA中需要实例化两个Student对象，在ActivityB中也需要一个Student对象，但我希望ActivityA中的两个Student都叫小明，但ActivityB中的Student对象叫小刚。

这样的需求，全局单例注解@Singleton是不行的，因为如果在AppModule中通过@Singleton注解提供Student，两个Activity中的Student都是小明（全局单例）了，而如果不用@Singleton注解，那么两个Activity中的Student是三个不同的对象，这样又不满足“ActivityA中的两个Student都叫小明”的需求。

这时我们就需要自定义@Scope注解实现局部单例了。

###1.自定义@Scope注解
实现方式很简单，首先这样自定义一个接口ActivityScope，声明这个注解可以使对象在同一个Activity中实现**单例**:

```
@Scope	//声明这是一个自定义@Scope注解
@Retention(RUNTIME)
public @interface ActivityScope {
}

```
###2.未使用@Scope注解效果展示：
**我们先声明2个Activity及相关代码：**
两个小明的Activity:

```
public class A03Activity extends AppCompatActivity {

    @BindView(R.id.btn_login)
    Button btnLogin;
    @BindView(R.id.tv_student1)
    TextView tvStudent1;
    @BindView(R.id.tv_student2)
    TextView tvStudent2;

    @Inject
    Student student1;
    @Inject
    Student student2;

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_a03);
        ButterKnife.bind(this);
        DaggerA03Component.builder()
                .a03Module(new A03Module(this))
                .build()
                .inject(this);

        //打印两个Student类
        tvStudent1.setText(student1.toString());
        tvStudent2.setText(student2.toString());
    }

    @OnClick(R.id.btn_login)
    public void onViewClicked() {
        startActivity(new Intent(this, A04Activity.class));
    }
}
```
小刚的Activity:

```
public class A04Activity extends AppCompatActivity {

    @BindView(R.id.tv_student)
    TextView tvStudent;

    @Inject
    Student student;

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_a04);
        ButterKnife.bind(this);
        DaggerA04Component.builder()
                .a04Module(new A04Module(this))
                .build()
                .inject(this);
		//打印Student
        tvStudent.setText(student.toString());
    }
}
```
相关的两个Module:

```
@Module
public class A03Module {

    private A03Activity activity;

    public A03Module(A03Activity activity) {
        this.activity = activity;
    }

    @Provides
    Student provideStudent() {
        return new Student();
    }
}

```

```
@Module
public class A04Module {

    private A04Activity activity;

    public A04Module(A04Activity activity) {
        this.activity = activity;
    }

    @Provides
    Student provideStudent() {
        return new Student();
    }

}
```
相关的Component:

```
@Component(modules = A03Module.class)
public interface A03Component {

    void inject(A03Activity activity);

}
```

```
@Component(modules = A04Module.class)
public interface A04Component {

    void inject(A04Activity activity);

}
```

编译成功后运行，结果如下：
![这里写图片描述](http://upload-images.jianshu.io/upload_images/7293029-cef3d66827ae8ea0?imageMogr2/auto-orient/strip)

可以看到，未使用自定义@Scope注解，每次@Inject注解获取的Student对象都是一个新的对象。

这样展示在界面的效果是3个不同的Student：

> ActivityA中的两个Student是小明和小红（并未实现Activity局部单例）
> ActivityB中的Student是小刚

###3.使用自定义@Scope注解：
下面我们稍作几行代码的改变，实现Activity局部单例：

####(1).在Module中添加@ActivityScope注解：

```
@Module
public class A03Module {

    private A03Activity activity;

    public A03Module(A03Activity activity) {
        this.activity = activity;
    }

    @Provides
    @ActivityScope//添加注解实现局部单例
    Student provideStudent() {
        return new Student();
    }

}
```
####(2).在Component中同样添加@ActivityScope注解，否则会编译报错：

```
@ActivityScope//添加注解实现局部单例
@Component(modules = A03Module.class, dependencies = AppComponent.class)
public interface A03Component {

    void inject(A03Activity activity);

}
```
好的，仅仅添加两行注解代码，我们接下来运行看结果：

![这里写图片描述](http://upload-images.jianshu.io/upload_images/7293029-3fa6da232a71378d?imageMogr2/auto-orient/strip)

成功！我们可以看到我们成功实现了这样的效果：
> ActivityA中的两个Student都是小明（Activity局部单例）
> ActivityB中的Student是小刚（换了Activity，生成另外一个对象）

也就是说,在添加@ActivityScope后，该Activity中通过@Inject依赖注入生成的Student对象全部唯一，但单例范围仅仅是该Activity中，出了该Activity，生成的Student仍然是非单例的。

## 三.@Singleton@Scope注解原理分析

这样实在太神奇了，我们不禁想到，仅仅通过这样一个声明，就能实现对象的生成和Activity的生命周期绑定吗？只要该Activity存在，里面的Student对象就是单例？

仅仅通过这样的声明,就能实现”同生共死“吗：

```
@Scope
@Retention(RUNTIME)	//Activity局部单例
public @interface ActivityScope {

}

```
```
@Scope
@Retention(RUNTIME)	//Fragment局部单例
public @interface FragmentScope {

}

```

分析源码前先公布答案：


> **自定义的@Singleton、@ActivityScope注解根本就没有这些功能,它的作用仅仅是”*标记*“。**

是不是难以置信，事实上的确如此，以小明的Activity为例，我们查看编译器为我们生成的源码目录：
![这里写图片描述](http://upload-images.jianshu.io/upload_images/7293029-05b2ee8a06a25c92?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

我们发现，无论是否添加@ActivityScope注解，自动生成的文件目录中文件的数量都没有改变，如上图，也就是说：

>**自定义@Scope注解并没有生成任何文件。**

那么自定义@Scope注解是如何实现单例的呢

####（1）A03Activity_MembersInjector.java

我们轻车熟路（如果您仔细阅读了前几篇文章的话）打开这个文件，发现**无论是否添加了@Scope注解，代码皆如下**：

```
public final class A03Activity_MembersInjector implements MembersInjector<A03Activity> {
  private final Provider<Student> student1AndStudent2Provider;

  public A03Activity_MembersInjector(Provider<Student> student1AndStudent2Provider) {
    assert student1AndStudent2Provider != null;
    this.student1AndStudent2Provider = student1AndStudent2Provider;
  }

  public static MembersInjector<A03Activity> create(Provider<Student> student1AndStudent2Provider) {
    return new A03Activity_MembersInjector(student1AndStudent2Provider);
  }

  @Override
  public void injectMembers(A03Activity instance) {
    if (instance == null) {
      throw new NullPointerException("Cannot inject members into a null reference");
    }
    //请注意该行
    instance.student1 = student1AndStudent2Provider.get();
    instance.student2 = student1AndStudent2Provider.get();
  }

  public static void injectStudent1(A03Activity instance, Provider<Student> student1Provider) {
    instance.student1 = student1Provider.get();
  }

  public static void injectStudent2(A03Activity instance, Provider<Student> student2Provider) {
    instance.student2 = student2Provider.get();
  }
}
```

需要注意的是，当我们依赖注入Activity时，最重要的其实是这两行代码：

```
  //请注意该行
  instance.student1 = student1AndStudent2Provider.get();
  instance.student2 = student1AndStudent2Provider.get();
```

无论是否添加了@ActivityScope注解，Activity中的俩个Student对象，都是通过student1AndStudent2Provider.get()进行实例化的，也就是说，当添加了@ActivityScope之后，get（）方法会生成单例，否则就没有单例，我们来看一下student1AndStudent2Provider对象是如何实例化的：

```
 private final Provider<Student> student1AndStudent2Provider;

 public A03Activity_MembersInjector(Provider<Student> student1AndStudent2Provider) {
 //2.构造方法中实例化student1AndStudent2Provider
    assert student1AndStudent2Provider != null;
    this.student1AndStudent2Provider = student1AndStudent2Provider;
  }

  public static MembersInjector<A03Activity> create(Provider<Student> student1AndStudent2Provider) {
  //1.有对象执行了该方法，然后执行构造方法
    return new 
    A03Activity_MembersInjector(student1AndStudent2Provider);
  }
```

可以看到student1AndStudent2Provider是通过有对象执行create（）方法，将student1AndStudent2Provider作为参数传进来进行的初始化，我们找到执行create（）方法的地方：**DaggerA03Component.java**

####（2）DaggerA03Component.java
通过对比DaggerA03Component.java，在Provider初始化的地方，我们发现了端倪：

a.未添加@ActivityScope:

```
  @SuppressWarnings("unchecked")
  private void initialize(final Builder builder) {
	//注意这行代码
    this.provideStudentProvider = A03Module_ProvideStudentFactory.create(builder.a03Module);

    this.a03ActivityMembersInjector = A03Activity_MembersInjector.create(provideStudentProvider);
  }

```

b.添加了@ActivityScope:

```
  @SuppressWarnings("unchecked")
  private void initialize(final Builder builder) {
   //注意这行代码，多了一个DoubleCheck.provider()方法
    this.provideStudentProvider =
        DoubleCheck.provider(A03Module_ProvideStudentFactory.create(builder.a03Module));

    this.a03ActivityMembersInjector = A03Activity_MembersInjector.create(provideStudentProvider);
  }

```

我们恍然大悟： **@ActivityScope注解改变的只是做一个标记，然后Component将有标记的对象工厂类（本文为Student工厂）进行了一次”DoubleCheck“单例加工！**
> 
> 我们有理由进行这样的猜测：
> 
> **1.未添加自定义@Scope**：每次Activity初始化对象，直接让工厂类初始化一个对象给Activty（activity.student = new Student()）;
> 
> **2.添加了自定义@Scope**：每次Activity初始化对象，直接让工厂类将单例对象给Activty（
> activity.student1 = factory.student(单例)；
> activity.student2 = factory.student(单例)；）;

是不是有一点恍然大悟的感觉？我们为了验证我们猜测的正确性，我们打开DoubleCheck类进行分析：

####（3*）DoubleCheck.java

```
public final class DoubleCheck<T> implements Provider<T>, Lazy<T> {
  private static final Object UNINITIALIZED = new Object();

  private volatile Provider<T> provider;
  private volatile Object instance = UNINITIALIZED;

  private DoubleCheck(Provider<T> provider) {
    assert provider != null;
    this.provider = provider;
  }

 // 单例获取对象
  @Override
  public T get() {
    Object result = instance;
    if (result == UNINITIALIZED) {
      synchronized (this) {
        result = instance;
        if (result == UNINITIALIZED) {
          result = provider.get();
          Object currentInstance = instance;
          if (currentInstance != UNINITIALIZED && currentInstance != result) {
            throw new IllegalStateException("Scoped provider was invoked recursively returning "
                + "different results: " + currentInstance + " & " + result);
          }
          instance = result;
          provider = null;
        }
      }
    }
    return (T) result;
  }
  
  public static <T> Provider<T> provider(Provider<T> delegate) {
    checkNotNull(delegate);
    if (delegate instanceof DoubleCheck) {
      return delegate;
    }
    return new DoubleCheck<T>(delegate);
  }

  public static <T> Lazy<T> lazy(Provider<T> provider) {
    if (provider instanceof Lazy) {
      @SuppressWarnings("unchecked")
      final Lazy<T> lazy = (Lazy<T>) provider;
    }
    return new DoubleCheck<T>(checkNotNull(provider));
  }
}
```
其中：
```
public final class DoubleCheck<T> implements Provider<T>, Lazy<T>
```


看到这行，我们明白了，其实和普通的Provider工厂类一样，DoubleCheck也是实现了Provider<T>接口，这样通过调用get（），就能获得对象的实例了，只不过稍微有所区别的是：

> 通过DoubleCheck.get（）获得的对象，是单例的。

至于DoubleCheck.get（）是如何实现单例的（似乎和我们普通实现单例的方式所有不同），篇幅所限，就不过多介绍，有兴趣的同学看看这篇博客 [多线程问题与double-check小结 @usher](http://blog.sina.com.cn/s/blog_597a437101011o66.html)，相信会有所收获。

## 总结

看到最后，相信大家已经对于@Scope和@Singleton的原理了解的差不多了，其实并没有什么魔力，能够真正通过一个注解，实现对Application生命周期或者Activity/Fragment生命周期进行绑定，无非做出一个标记的作用，检查到标记后，Component将对应的工厂实现单例模式，这样我们的容器去获取对象时就能实现对应的单例效果。

而对于其他的Activity，因为ActivityA和ActivityB是同一层级互不干扰的，ActivityA的Component和ActivityB的Component互不联系，当然不可能相互共享单例：

![这里写图片描述](http://upload-images.jianshu.io/upload_images/7293029-6dd2564f087362fd.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

> 借用[@MarkZhai 大神的图和一句话：](http://blog.zhaiyifan.cn/2016/03/27/android-new-project-from-0-p4/)位于上层的component是看不到下层的，而下层则可以使用上层的，但不能引用同一层相邻component内的实例。

同时，当我们的容器（Application/Activity/Fragment等）销毁时，对应的Component当然也不复存在，这就是为什么我们看起来@Scope能够实现”同生共死“效果的原因。

至此，Dagger2相关的知识点我们已经了解的差不多了，我们需要的就是更多的去使用它，通过它实现快速开发和代码解耦。

[代码已托管Github，源码传送门，点我查看](https://github.com/qingmei2/Sample_dagger2)

## 2017/8/24日更新

 事实上，Android开发中使用Dagger,开发人员仍然需要面对一些问题。

google工程师们尝试弥补Dagger的问题，于是Dagger2-Android,基于Dagger2，应用于Android开发，由google开发的的拓展库应运而生：

* [ 告别Dagger2模板代码：DaggerAndroid使用详解 ](http://www.jianshu.com/p/917bf39cae0d)
* [ 告别Dagger2模板代码：DaggerAndroid原理解析 ](http://www.jianshu.com/p/d4d62945d9c8)
