#Android 神兵利器Dagger2使用详解（一）基础篇

本系列书写原因：在公司一个新的共同开发项目中，使用到了Dagger2依赖注入，在使用它的时候，因为框架的原因产生了一些问题（代码风格的不同？），发现自己对于Dagger2还是有一些没有理解到位的地方，于是干脆抽个时间搞懂它，从最基础的使用开始，我们一点点从**源码**深入它，去感受依赖注入可以给代码开发带来怎样的魅力。

### 本系列所有文章：

[ Android 神兵利器Dagger2使用详解（一）基础使用 ](http://www.jianshu.com/p/b40bcd1a9ec9)
[ Android 神兵利器Dagger2使用详解（二）Module&Component源码分析](http://www.jianshu.com/p/30d48ddefd30)
[ Android 神兵利器Dagger2使用详解（三）MVP架构下的使用](http://www.jianshu.com/p/c46acc3f21ab)
[ Android 神兵利器Dagger2使用详解（四）Scope注解的使用及源码分析 ](http://www.jianshu.com/p/caaac320c785)
[ 告别Dagger2模板代码：DaggerAndroid使用详解 ](http://www.jianshu.com/p/917bf39cae0d)
[ 告别Dagger2模板代码：DaggerAndroid原理解析 ](http://www.jianshu.com/p/d4d62945d9c8)
该系列首发于我的CSDN专栏 ：
[ Android开发：Dagger2详解 ](http://blog.csdn.net/column/details/17168.html)

##1 什么是依赖注入

依赖注入是一种面向对象的编程模式，它的出现是为了降低耦合性，所谓耦合就是类之间依赖关系，所谓降低耦合就是降低类和类之间依赖关系。可能有的人说自己之前并没有使用过依赖注入，其实真的没有使用过吗？当我们在一个类的构造函数中通过参数引入另一个类的对象，或者通过set方法设置一个类的对象其实就是使用的依赖注入,比如：

```
//简单的依赖注入，构造方法或者set（）方法都属于依赖注入
public class ClassA {
    ClassB classB;
    public void ClassA(ClassB b) {
        classB = b;
    }
}
```
另外还有一种方式可以作为依赖注入，那就是通过注解的方式注入，Dagger2就是通过注解的方式完成依赖注入的。

**使用方式1**，添加依赖(在module的build.gradle中添加如下代码)

```
  compile 'com.google.dagger:dagger:2.7'
  annotationProcessor 'com.google.dagger:dagger-compiler:2.7'
```
**使用方式2**，最新的AndroidStudio在使用apt插件的时候已经会报warn了，但并不是不能使用，我们也可以通过apt插件使用Dagger2：

在你的Project build.gradle中添加如下代码
```
dependencies {
    classpath 'com.android.tools.build:gradle:2.3.1'
    //添加apt插件
    classpath 'com.neenbedankt.gradle.plugins:android-apt:1.8'
 }
```
然后添加依赖(在module的build.gradle中添加如下代码)
```
apply plugin: 'com.android.application'
apply plugin: 'com.neenbedankt.android-apt'

	...
dependencies {
	...
compile 'com.google.dagger:dagger:2.7'
apt 'com.google.dagger:dagger-compiler:2.7'
compile 'org.glassfish:javax.annotation:10.0-b28'
	...
}
```

##二 基本使用
Dagger2最好的搭配是通过MVP框架进行开发，但是我们不急，先从最简单的一个案例开始入手。

###1 我们先创建一个简单的Student类：

```
public class Student {

    @Inject
    public Student() {
    }
}
```
非常简单，没有任何属性，Student类中只有一个空的构造方法，不同以往的是，我们在构造方法上面添加了一个@Inject注解，这个注解有什么用呢？

我们使用ctrl+F9(mac使用Cmd+F9)进行一次编译，几秒后编译结束，似乎没有发生任何事情......

那当然是不可能的，我们打开app下的路径，我的路径是：app\build\generated\source\apt\debug\com\mei_husky\sample_dagger2\model\Student_Factory.java

唉，我们发现，似乎编译器帮我们自动生成了一个文件，叫做Student_Factory类，我们点击进去：

```
@Generated(
  value = "dagger.internal.codegen.ComponentProcessor",
  comments = "https://google.github.io/dagger"
)
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
代码并不难理解，似乎是一个工厂类，在通过create（)创建后，每次调用get()方法都能获得一个Student对象。

我们似乎明白了点什么，原来我们通过@Inject注解了一个类的构造方法后，可以让编译器帮助我们产生一个对应的Factory类，通过这个工厂类我们可以通过简单的get()方法获取到Student对象！

##2.创建一个Activity调用Student：

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
    }

    @OnClick(R.id.btn_01)
    public void onViewClicked(View view) {
        switch (view.getId()){
            case R.id.btn_01:             Toast.makeText(this,student.toString(),Toast.LENGTH_SHORT).show();
            break;
        }
    }
}
```
接下来我们创建一个Activity类，在这个类中创建一个成员变量Student，按照Dagger2给我们的指示，当我们需要一个Student，我们只需要在这个成员变量上方加一个@Inject注解，编译器会自动帮我们产生对应的代码，我们就可以直接使用这个Student对象了！

本案例中我们设置一个Button，点击Button后我们打印出这个Student对象。

事实真的如此吗？我们直接运行代码，并点击Button，很遗憾，直接报空指针异常：

![这里写图片描述](http://upload-images.jianshu.io/upload_images/7293029-9c33c3555c387735?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

显然，和平常使用的结果一样，@Inject并没有帮助我们初始化对应的Student对象，或者说，我们的Activity并没有使用刚才我们看到的Student_Factory类，不过也可以理解，我们并没有建立Activity和Student_Factory类之间的关系嘛。

##3. 曲线救国，Component接口
我们接下来创建一个Module类以及一个Component接口：

```
@Module
public class A01SimpleModule {

    private A01SimpleActivity activity;

    public A01SimpleModule(A01SimpleActivity activity) {
        this.activity = activity;
    }
}
```

```
@Component(modules = A01SimpleModule.class)
public interface A01SimpleComponent {

    void inject(A01SimpleActivity activity);

}
```
请注意，Module类上方的@Module注解意味着这是一个提供数据的【模块】，而Component接口上方的@Component(modules = A01SimpleModule.class)说明这是一个【组件】（我更喜欢称呼它为注射器）。

突然出现的这两个类可以称得上是莫名其妙，因为我们从代码上来看并不知道这对于Student和Activity之间关系有什么实质性的进展，但假如我们这时在Activty中添加这一段代码：

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

然后运行代码点击Button，神奇的事情发生了：

![这里写图片描述](http://upload-images.jianshu.io/upload_images/7293029-081e7d392f00ef67?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

显然，添加了两个看起来莫名其妙的Module和Component类，然后在Activity中添加一段代码，被@Inject的Student类被成功**依赖注入**到了Activity中，我们接下来就可以肆无忌惮使用这个Student对象了！

## 三 我们为什么使用依赖注入
这时候不可避免的，有些同学会有些疑问，我们为什么要花费这么大的力气（时间成本）去学习这样一个看起来很鸡肋的Dagger呢？我们需要一个Student对象，完全可以直接通过new的方式创建一个嘛！

当然是有必要的，因为通常简单的代码具有耦合性，而要想降低这样的耦合就需要其他的辅助代码，其实少代码量和低耦合这两者并不能同时兼顾。

试想，我们如果通过这样的方式，在其他的文件中创建了这样若干个（心大一些，我们是一个大项目的唯一负责人），不，1000个文件中使用到了Student对象，我们至少要new 1000个新的Student类对象，这时，新的需求到了，我们的Student需要添加一个String类型的参数name。

what the fuck？ 这意味我需要分别跑这1000个文件中逐个修改new Student()的那行代码吗？

如果是Dagger2，当然不需要，我们只需要在Student类中做出简单的修改即可，这在后文中将会提到（因为涉及参数问题），我们只需要知道只需轻松几步即可完成Student的构造修改问题，达到低耦合的效果即可。

## 四 神奇的Module和Component作用详解

现在我们具体的思考以下一个场景：

我们假设案例中的Activity代表家庭住址，Student代表某个商品，现在我们需要在家（Activity）中使用商品（Student），我们网购下单，商家（代表着案例中自动生成的Student_Factory工厂类）将商品出厂，这时我们能够在家直接获得并使用商品吗？

当然不可能，虽然商品（Student）已经从工厂（Factory）生产出来，但是并没有和家（Activity）建立连接，我们还需要一个新的对象将商品送货上门，这种英雄级的人物叫做——快递员（Component,注入器）。

没错，我们需要这样的一种注入器，将已经生产的Student对象传递到需要使用该Student的容器Activity中，于是我们需要在Activity中增加这样几行代码：

```
  //新添代码
  DaggerA01SimpleComponent.builder()
          .a01SimpleModule(new A01SimpleModule(this))
          .build()
          .inject(this);
```
这就说明快递员Component已经将对象Inject（注入）到了this（Activity）中了，既然快递到家，我们当然可以直接使用Student啦！

这时有朋友可能会问，原来Component起的作用是这样，那么Module是干嘛的呢？

事实上，在这个案例中，我们将这行代码进行注释后运行，发现我们依然可以使用Student对象：

```
        //新添代码
        DaggerA01SimpleComponent.builder()
              //.a01SimpleModule(new A01SimpleModule(this))//注释掉这行代码
                .build()
                .inject(this);
```

我们可以暂时这样理解Module，它的作用就好像是快递的箱子，里面装载的是我们想要的商品，我们在Module中放入什么商品，快递员（Component）将箱子送到我们家（Activity容器），我们就可以直接使用里面的商品啦！

为了展示Module的作用，我们重写Student类，**取消了@Inject注解**：

```
public class Student {
    
    public Student() {
    }

}

```

Module中添加这样一段代码：

```
@Module
public class A01SimpleModule {

    private A01SimpleActivity activity;

    public A01SimpleModule(A01SimpleActivity activity) {
        this.activity = activity;
    }
    //下面为新增代码：
    @Provides
    Student provideStudent(){
        return new Student();
    }
}
```
然后Component不变，Activity中仍然是这样：

```
 DaggerA01SimpleComponent.builder()
                .a01SimpleModule(new A01SimpleModule(this))
                .build()
                .inject(this);
```
然后运行，我们发现，仍然可以点击使用Student对象！

原因很简单，虽然@Inject注解取消了，但是我们已经在快递箱子（Module）中通过@Providers放入了一个Student对象，然后让快递员（Component）送到了家中（Activity），我们当然可以使用Student对象了！

##五 小结

经过简单的使用Dagger2，我们已经可以基本有了以下了解：

> **@Inject ：** 注入，被注解的构造方法会自动编译生成一个Factory工厂类提供该类对象。
> 
> **@Component:** 注入器，类似快递员，作用是将产生的**对象**注入到需要对象的**容器**中，供容器使用。
> 
> **@Module:** 模块，类似快递箱子，在Component接口中通过@Component(modules =
> xxxx.class),将容器需要的商品封装起来，统一交给快递员（Component），让快递员统一送到目标容器中。

现在看回来，我们仅仅通过几个注解就能使用对象，还是很方便的，我会在接下来的文章中对@Component和@Module注解之后，编译期生成的代码进行解析，看看到底这两个注解到底起着什么样的作用。

[GitHub传送门，点我看源码](https://github.com/qingmei2/Sample_dagger2)

[ Android 神兵利器Dagger2使用详解（二）Module&Component源码分析](http://www.jianshu.com/p/30d48ddefd30)
