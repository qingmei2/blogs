![](http://upload-images.jianshu.io/upload_images/7293029-231dfa44a754a00a.jpeg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

##前言：
### 本系列所有文章：

[ Android 神兵利器Dagger2使用详解（一）基础使用 ](http://www.jianshu.com/p/b40bcd1a9ec9)
[ Android 神兵利器Dagger2使用详解（二）Module&Component源码分析](http://www.jianshu.com/p/30d48ddefd30)
[ Android 神兵利器Dagger2使用详解（三）MVP架构下的使用](http://www.jianshu.com/p/c46acc3f21ab)
[ Android 神兵利器Dagger2使用详解（四）Scope注解的使用及源码分析 ](http://www.jianshu.com/p/caaac320c785)
[ 告别Dagger2模板代码：DaggerAndroid使用详解 ](http://www.jianshu.com/p/917bf39cae0d)
[ 告别Dagger2模板代码：DaggerAndroid原理解析 ](http://www.jianshu.com/p/d4d62945d9c8)
该系列首发于我的CSDN专栏 ：
[ Android开发：Dagger2详解 ](http://blog.csdn.net/column/details/17168.html)

在我的上一篇文章中，我们通过Dagger2依赖注入的两种方式获取Student对象,并简单了解了各个组件的作用和互相的联系:

> @Inject ： 注入，被注解的构造方法会自动编译生成一个Factory工厂类提供该类对象。

>@Component: 注入器，类似快递员，作用是将产生的对象注入到需要对象的容器中，供容器使用。

>@Module: 模块，类似快递箱子，在Component接口中通过@Component(modules = 
xxxx.class),将容器需要的商品封装起来，统一交给快递员（Component），让快递员统一送到目标容器中。

本文我们继续按照上文案例来讲，通过源码分析，看看究竟是为什么，我们能够仅仅通过数个注解，就能随心所欲使用Student对象。

## 一 .代码回顾
我们先不考虑Module，还是这样的代码：
###1 .Student类
```
public class Student {

    @Inject
    public Student() {
    }

}

```
###2 .Module类

```
@Module
public class A01SimpleModule {

    private A01SimpleActivity activity;

    public A01SimpleModule(A01SimpleActivity activity) {
        this.activity = activity;
    }

}
```

###3.Component类

```
@Component(modules = A01SimpleModule.class)
public interface A01SimpleComponent {

    void inject(A01SimpleActivity activity);

}
```

###4.Activity类

```
public class A01SimpleActivity extends AppCompatActivity {

    @BindView(R.id.btn_01)
    Button btn01;

    @Inject
    Student student;

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_a01_simple);
        ButterKnife.bind(this);
        //新添代码
        DaggerA01SimpleComponent.builder()
//                .a01SimpleModule(new A01SimpleModule(this))
                .build()
                .inject(this);
    }

    @OnClick(R.id.btn_01)
    public void onViewClicked(View view) {
        switch (view.getId()){
            case R.id.btn_01:
                Toast.makeText(this,student.toString(),Toast.LENGTH_SHORT).show();
                break;
        }
    }

```

然后运行代码，点击Button，输出student对象信息：

![这里写图片描述](http://upload-images.jianshu.io/upload_images/7293029-e1197561db783e51?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

##二.源码解析

我们打开app目录下的build文件夹，以笔者为例，目录结构为：

>app\build\generated\source\apt\debug\......\A01SimpleActivity_MembersInjector.java

我们不难发现，编译器已经帮我们生成了这样几个文件：

> DaggerA01SimpleComponent.java
>  Student_Factory.java
> A01SimpleActivity_MembersInjector.java

我们进行一一分析：

### 1. Student_Factory.java
上一篇文章我们已经进行了分析，很简单，当我们@Inject注解一个类的构造方法时，编译器会自动帮我们生成一个工厂类，负责生产该类的对象，**类似于商品的厂家**：

```
public enum Student_Factory implements Factory<Student> {
  INSTANCE;

  @Override
  public Student get() {
    return new Student();
  }

  public static Factory<Student> create() {
    return INSTANCE;
  }
}
```

###2.DaggerA01SimpleComponent.java

```
public final class DaggerA01SimpleComponent implements A01SimpleComponent {
  private MembersInjector<A01SimpleActivity> a01SimpleActivityMembersInjector;

  private DaggerA01SimpleComponent(Builder builder) {
    assert builder != null;
    initialize(builder);
  }

  public static Builder builder() {
    return new Builder();
  }

  public static A01SimpleComponent create() {
    return builder().build();
  }

  @SuppressWarnings("unchecked")
  private void initialize(final Builder builder) {
	//初始化A01SimpleActivity_MembersInjector
    this.a01SimpleActivityMembersInjector =
        A01SimpleActivity_MembersInjector.create(Student_Factory.create());
  }

  @Override
  public void inject(A01SimpleActivity activity) {
    a01SimpleActivityMembersInjector.injectMembers(activity);
  }

  public static final class Builder {
    private Builder() {}

    public A01SimpleComponent build() {
      return new DaggerA01SimpleComponent(this);
    }
   /**
     * @deprecated This module is declared, but an instance is not used in the component. This method is a no-op. For more, see https://google.github.io/dagger/unused-modules.
     */
    @Deprecated
    public Builder a01SimpleModule(A01SimpleModule a01SimpleModule) {
      Preconditions.checkNotNull(a01SimpleModule);
      return this;
    }
  }
}
```
很熟悉，我们在Activity中就用到了这个生成的类，编译器起名方式也很简洁：Dagger+你的Component接口名。

在我们的Activity中我们是这样使用：

```
DaggerA01SimpleComponent.builder()
//              .a01SimpleModule(new A01SimpleModule(this))
                .build()
                .inject(this);
```

我们根据这个步骤查看源码，发现

> DaggerA01SimpleComponent.builder().build()

实际上是通过建造者模式创建了一个新的DaggerA01SimpleComponent对象，在这个对象的构造方法中，执行了initialize()方法，初始化了一个A01SimpleActivity_MembersInjector对象。

请注意，在初始化A01SimpleActivity_MembersInjector时我们看到这行代码：

>A01SimpleActivity_MembersInjector.create(Student_Factory.create());

**可以看到，Student工厂类作为参数传入了Injector中。**

然后通过调用

>  DaggerA01SimpleComponent.builder().build().**inject(this);**

中，实际上是将Activity作为参数传入了A01SimpleActivity_MembersInjector对象的InjectMembers()方法里面，仅此而已。

很好，我们看起来已经明白了Component的作用：编译器通过@Component注解，生成了DaggerA01SimpleComponent类，然后将activity传入初始化了的A01SimpleActivity_MembersInjector对象中。

这时我们有了一点头绪，因为我们发现，Student工厂类，已经和Activity同时都放入了这个神秘的A01SimpleActivity_MembersInjector类中了。

###3.A01SimpleActivity_MembersInjector类,将Student和Activity进行连接

```
public final class A01SimpleActivity_MembersInjector implements MembersInjector<A01SimpleActivity> {
  private final Provider<Student> studentProvider;

  public A01SimpleActivity_MembersInjector(Provider<Student> studentProvider) {
    assert studentProvider != null;
    this.studentProvider = studentProvider;
  }

  public static MembersInjector<A01SimpleActivity> create(Provider<Student> studentProvider) {
    return new A01SimpleActivity_MembersInjector(studentProvider);
  }

  @Override
  public void injectMembers(A01SimpleActivity instance) {
    if (instance == null) {
      throw new NullPointerException("Cannot inject members into a null reference");
    }
    instance.student = studentProvider.get();
  }

  public static void injectStudent(A01SimpleActivity instance, Provider<Student> studentProvider) {
    instance.student = studentProvider.get();
  }
}
```
其实已经很简单了，在该Injector的injectMembers()方法中，已经将Student对象通过Student_Factory的get()方法获得，然后直接赋值给Activity的student对象了！

就是这行代码：

> instance.student = studentProvider.get();

 private final Provider<Student> studentProvider ->就是在create（）方法中传入的Student_Factory工厂类，不信？点击Factory类：

```
public interface Factory<T> extends Provider<T> {
}
```
很明显了，Student_Factory父类是 Factory,Factory父类是Provider，向上转型嘛。

## 三 带Module的源码解析：

现在我们尝试上一篇文章中的Module相关代码:

###1.Student类（取消Inject注解）：

```
public class Student {

    public Student() {
    }

}
```
###2.Module类（增加一个Provide注解方法）：

```
@Module
public class A01SimpleModule {

    private A01SimpleActivity activity;

    public A01SimpleModule(A01SimpleActivity activity) {
        this.activity = activity;
    }

    @Provides
    Student provideStudent(){
        return new Student();
    }
}

```

###3.Component(不变)

```
@Component(modules = A01SimpleModule.class)
public interface A01SimpleComponent {

    void inject(A01SimpleActivity activity);

}

```

###4.Activity（新增一行代码）

```
public class A01SimpleActivity extends AppCompatActivity {

    @BindView(R.id.btn_01)
    Button btn01;

    @Inject
    Student student;

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_a01_simple);
        ButterKnife.bind(this);
        //新增一行代码.a01SimpleModule(new A01SimpleModule(this))
        DaggerA01SimpleComponent.builder()
                .a01SimpleModule(new A01SimpleModule(this))
                .build()
                .inject(this);
    }

    @OnClick(R.id.btn_01)
    public void onViewClicked(View view) {
        switch (view.getId()){
            case R.id.btn_01:
                Toast.makeText(this,student.toString(),Toast.LENGTH_SHORT).show();
                break;
        }
    }
}

```
我们**先把app/build文件夹删除**，删除自动生成的代码后，然后ctrl+F9重新编译,编译成功运行，依然可以获得Student对象。


这时我们打开build目录，层层剥开后，发现这样三个类：

> DaggerA01SimpleComponent.java
> A01SimpleModule_ProvideStudentFactory.java
> A01SimpleActivity_MembersInjector.java

###1.A01SimpleModule_ProvideStudentFactory.java


```
public final class A01SimpleModule_ProvideStudentFactory implements Factory<Student> {
  private final A01SimpleModule module;

  public A01SimpleModule_ProvideStudentFactory(A01SimpleModule module) {
    assert module != null;
    this.module = module;
  }

  @Override
  public Student get() {
    return Preconditions.checkNotNull(
        module.provideStudent(), "Cannot return null from a non-@Nullable @Provides method");
  }

  public static Factory<Student> create(A01SimpleModule module) {
    return new A01SimpleModule_ProvideStudentFactory(module);
  }
}
```

我们知道，我们在Module中创建了一个provideStudent()方法，方法中创建并返回了一个Student对象，其实很相似，Module的@Provides注解就是帮助我们生成了一个Student_Factory的工厂，只不过这个工厂很特别，只有钥匙才能进(必须传入A01SimpleModule对象才能实例化)：

```
//没有传入A01SimpleModule对象，无法实例化该工厂类
 public static Factory<Student> create(A01SimpleModule module) {
    return new A01SimpleModule_ProvideStudentFactory(module);
  }
```

我们查看A01SimpleModule会发现，想实例化A01SimpleModule，必须传入一个A01SimpleActivity对象，这说明了，**A01SimpleModule就像是一个专属的快递箱子，只有本人（A01SimpleActivity）才能签收私人快递，然后打开自己的盒子（A01SimpleModule->创建A01SimpleModule_ProvideStudentFactory）获得这个Student对象！**

简单来说，通过@Providers注解后，产生的对象就经过Module包装，通过Component快递员送到需要的容器Activity中。

相比@Inject简单粗暴的注解生成的“万能工厂”Student_Factory类，似乎这个更“安全”一些~

###2.DaggerA01SimpleComponent 

```
public final class DaggerA01SimpleComponent implements A01SimpleComponent {
  private Provider<Student> provideStudentProvider;

  private MembersInjector<A01SimpleActivity> a01SimpleActivityMembersInjector;

  private DaggerA01SimpleComponent(Builder builder) {
    assert builder != null;
    initialize(builder);
  }

  public static Builder builder() {
    return new Builder();
  }

  @SuppressWarnings("unchecked")
  private void initialize(final Builder builder) {
	//创建A01Module专属的工厂
    this.provideStudentProvider =
        A01SimpleModule_ProvideStudentFactory.create(builder.a01SimpleModule);
	//将专属工厂放入Injector中
    this.a01SimpleActivityMembersInjector =
        A01SimpleActivity_MembersInjector.create(provideStudentProvider);
  }

  @Override
  public void inject(A01SimpleActivity activity) {
	  //将Activity容器传入Injector中
    a01SimpleActivityMembersInjector.injectMembers(activity);
  }

  public static final class Builder {
    private A01SimpleModule a01SimpleModule;

    private Builder() {}

    public A01SimpleComponent build() {
      if (a01SimpleModule == null) {
        throw new IllegalStateException(A01SimpleModule.class.getCanonicalName() + " must be set");
      }
      return new DaggerA01SimpleComponent(this);
    }
	
	//传入需要的Module
    public Builder a01SimpleModule(A01SimpleModule a01SimpleModule) {
      this.a01SimpleModule = Preconditions.checkNotNull(a01SimpleModule);
      return this;
    }
  }

```

变化很少，相比较@Inject注解的方式，@Inject注解生成的工厂类就好像将商品赤裸着交给Component,@module注解生成的工厂类就好像给商品加了一层防护纸箱，感觉更贴心了呢~

###3.A01SimpleActivity_MembersInjector 

```
public final class A01SimpleActivity_MembersInjector implements MembersInjector<A01SimpleActivity> {
  private final Provider<Student> studentProvider;

  public A01SimpleActivity_MembersInjector(Provider<Student> studentProvider) {
    assert studentProvider != null;
    this.studentProvider = studentProvider;
  }

  public static MembersInjector<A01SimpleActivity> create(Provider<Student> studentProvider) {
    return new A01SimpleActivity_MembersInjector(studentProvider);
  }

  @Override
  public void injectMembers(A01SimpleActivity instance) {
    if (instance == null) {
      throw new NullPointerException("Cannot inject members into a null reference");
    }
    instance.student = studentProvider.get();
  }

  public static void injectStudent(A01SimpleActivity instance, Provider<Student> studentProvider) {
    instance.student = studentProvider.get();
  }
}
```

可以发现，基本并没有什么变化。

## 总结

经过两次分析 我们基本理解了Dagger2的使用方式，原理基本如下：

@Inject 注解构造  生成“大众”工厂类			
**或者**
@Module +@Providers 提供注入“私有”工厂类

然后

通过Component 创建获得Activity,获得工厂类Provider，统一交给Injector

最后

Injector将Provider的get()方法提供的对象，注入到Activity容器对应的成员变量中，我们就可以直接使用Activity容器中对应的成员变量了！

了解了原理，接下来怎么使用就随意了~在接下来的文章中，我会结合MVP的架构对Dagger2进行更深入的使用。

[GitHub传送门，点我看源码](https://github.com/qingmei2/Sample_dagger2)
[ Android 神兵利器Dagger2使用详解（三）MVP架构下的使用](http://www.jianshu.com/p/c46acc3f21ab)
