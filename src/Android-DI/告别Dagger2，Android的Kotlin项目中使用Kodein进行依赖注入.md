## 前言：依赖注入浅谈
---

### Dagger2的困境

对于**依赖注入**（Dependency Injection，简称DI）来讲，它并非是一个新鲜的词汇，实际上，它很早就被提出并且应用在了企业级的web应用开发当中，比如Spring。

在Android开发领域内，毫无疑问，Google大名鼎鼎的 **[Dagger2](https://github.com/google/dagger)** 是依赖注入框架的首选工具库，它非常优秀，Github上数以万计的star是最强力的佐证，但是缺点也很明显，那就是：

**复杂**。

这最直接导致了 **极为高昂的学习成本**——你可以不认为它难，但你要承认学习**[Dagger2](https://github.com/google/dagger)**比你学习其他库所花费的时间要多得多。

### 我的感受

我是[却把清梅嗅](https://github.com/qingmei2)，一个Android开发者，此外在业余时间，我喜欢学习总结分享。

对于Dagger，我曾经花了6篇博客的篇幅去阐述了我的理解（请参考https://blog.csdn.net/column/details/17168.html），于现在来看，这些文章还存在一些瑕疵，但对于当时的我来说，我尽力去阐述了，但是还有问多评论这样说：

> 文章讲的很好，但是我还是遇到了一些问题，请教博主........

Dagger太复杂了！一方面它意味着**更灵活，更多强大功能**的支持，同时它也意味着——不够简洁。

你需要配置Module，需要配置Component，然后在你需要注入的容器中，初始化Component然后注入进来——这一切的前提是，你还得先正确地进行依赖配置，以保证编译器不会爆出一大串Error。

当然，对于其中的一些问题，世界上那些最顶尖的工程师们已经做出了更好的解决方案，**dagger.android横空出世**，对于Activity和Fragment容器，开发者再也不需要初始化Component的模版代码了。

dagger.android并没有解决 **学习成本高昂**的问题——相反，它需要在熟悉Dagger2的基础上继续深入，反而增加了学习成本；但不可否认，它依然是目前Android开发中依赖注入工具选型中的首选。

我也曾经一度这样想过，对于依赖注入这个技能点，**熟练dagger.android的使用，阅读源码并掌握原理** 已经足够。

**——但Kotlin时代到来了。**

### Kotlin时代

最近个人在尝试构建适合自己的 **Kotlin**的 MVVM ，在依赖注入框架的选型上，我最终选择了 **[Kodein](https://github.com/Kodein-Framework/Kodein-DI)** 。

这是一个非常轻量级的DI框架，同样，它也被**《Kotlin in Action》**一书所推荐，相比于配置繁琐的Dagger，它的配置过程更清晰且简单，并且，这个库的源码也是 **Kotlin** 的。

有同学说，虽然 **Dagger** 配置很繁琐，但 **dagger.android** 已经大大减少了模板代码，为什么不使用它呢？

确实如此，但**Dagger**终究是通过编译器自动生成 **Java** 代码的库，这实在不够 **Kotlin**，于我个人来讲，**Dagger**并非最优选。

## Kodein：入门篇

---

虽然 Kodein 全名为 **KO**tlin **DE**pendency **IN**jection，但Kodein并不是一个真正的依赖注入框架。 他们的官方文档将其称作**依赖检索容器**。

下面是Kodein的官方文档：

[Kodein官方文档：Getting started with Kodein DI](http://kodein.org/Kodein-DI/?5.2/getting-started#_the_type_erasure_problem)
[Kodein官方文档 for Android：Kodein DI on Android](http://kodein.org/Kodein-DI/?5.2/android)

本文的主旨是，让开发者更快入门Kodein和理解其思想，如果您想更深入学习它，**官方文档**是你不二的选择。

> 在这里我推荐官方文档的原因，一是，英文文档对一些专业词汇和思想，描述的更清晰准确；第二就是，关于Kodein目前国内还没有任何相关中文资料....

让我们开始使用它吧。

### 1.添加依赖

在Module级别的build.gradle中添加Kodein最新的依赖：

```groovy
// 基础组件
implementation 'org.kodein.di:kodein-di-generic-jvm:5.2.0'
// Android扩展组件
implementation 'org.kodein.di:kodein-di-framework-android-core:5.2.0'
// support扩展组件，我的项目中用到了v4包的Fragment，因此我需要它
implementation 'org.kodein.di:kodein-di-framework-android-support:5.2.0'
```

如果依赖不成功，你需要把kodein的maven仓库显式地声明出来：

```groovy
allprojects {
    repositories {
        google()
        jcenter()
        // maven for kodein
        maven { url 'https://dl.bintray.com/kodein-framework/Kodein-DI/' }
    }
}
```

### 2.如何使用它？

我们必须尝试回忆Dagger2的核心思想：它是将依赖通过Module管理提供，然后交给Component注入给Activity等容器。

> Kodein并不是一个真正的依赖注入框架。 他们的官方文档将其称作**依赖检索容器**。

这是我文中第二次声明，Kodein和Dagger的核心思想有所不同，其原理是——将依赖交给一个 `Kodein容器`，然后将`Kodein容器`交给`Activity`，`Activity`中所需要的依赖通过**委托**给`Kodein容器`注入。

好了好了，我知道这段话很抽象，我们来看一个案例，它将展示如何把一个`SQLiteDatabase`对象通过`Kodein`进行依赖注入。

首先我们先声明一个Kodein容器：

```kotlin
val kodein = Kodein {
    bind<Database>() with singleton { SQLiteDatabase() }
}
```

Kodein提供了对DSL的强大支持，正如你所看到的，我们可以将`SQLiteDatabase`对象的实例化过程，放在Kodein 开头的`{ }`中。

`bind<T>() `意味着你声明将一个类型为`T`的依赖放入了Kodein容器进行绑定（bind）。作为一个非常重的对象，`SQLiteDatabase`更应该保持**单例**，所以我们在对其实例化的方式上，选择了`singleton { }`。

相比于`dagger`,这种配置方式实在太清晰了——没有`@Inject`，没有`@Providers`，没有`@Component`,你只需要通过Kotlin所支持的DSL，就能轻松完成各种方式依赖的绑定。

### 3.有哪些绑定方式呢？

本文不是Kodein的文档，但是我认为花一些篇幅讲解这些是有必要的，Kodein共提供了`provider, singleton, factory, multiton, instance`等等多种方式的绑定。

#### singleton 

正如上文描述过的，这种方式会实例化一个**单例**对象，该单例对象将会在第一次使用时通过单例函数进行创建，该函数不带参数并返回绑定类型的对象（例如`（）→T`）。

示例代码，该对象将会在第一次被调用时，通过调用该函数，将对象进行生成并返回，该函数有且仅会有一次被调用：

```kotlin
val kodein = Kodein {
    bind<DataSource>() with singleton { SqliteDS.open("path/to/file") }
}
```

#### provider

和singleton不同，该函数每次都会被调用并返回对应的依赖。

示例代码,每次都会调用该函数，返回一个新生成的对象：

```kotlin
val kodein = Kodein {
    bind<Die>() with provider { RandomDie(6) }
}
``` 

#### factory

和`provider`很相似，每次都会调用该函数，返回一个新生成的对象，不同的是，`factory`函数接受已定义类型的参数并返回绑定类型的对象（例如，`（A）→T`）。

示例代码,根据参数`sides`的不同，每次都会返回一个新的`Die`：

```kotlin
val kodein = Kodein {
    bind<Die>() with factory { sides: Int -> RandomDie(sides) }
}
```

#### 还有更多...

还有更多，请参考官方文档中[关于绑定声明方式的说明](https://kodein.org/Kodein-DI/?5.2/core#declaring-dependencies)。

### 4.进行依赖注入

我们已经通过不同的方式，完成了依赖绑定，接下来，我们就可以进行依赖注入了。

还记得我已经提了两遍的话吗，Kodein是一个**依赖检索容器**。

我们把绑定的依赖交给`Kodein`容器，然后我们把这个容器交给`Activity`，`Activity`就可以从容器中取出这些依赖了。

稍微有点不同的是，取出依赖的方式是通过kotlin的`属性委托`：

```Kotlin
class Presenter(val kodein: Kodein) {  // presenter拥有kodein容器
    private val db: Database by kodein.instance()  // 通过属性委托，即可依赖注入
    private val rnd: Random by kodein.instance()
}
```

现在，我们就可以直接对这些对象进行引用了，have fun！

## Kodein：实战篇
---
> **抛开项目架构谈工具都是刷流氓**。


上述内容仅仅提供了对Kodein的简单了解，实际上，无论是是`dagger2`还是`kodein`，**会写demo** 和 **在项目中应用** 完全是天差地别。

如果只是简单的API介绍，这篇文章也许更早就出来了，事实上，我在尝试构建 **Kotlin**的 MVVM 项目时，将Kodein加了进去，并不断进行调整——直到现在，我对它有了更清晰的一些认识以及理解。

当然，它们不一定就是对的，或者说，不一定就是适合你的，但我希望，我的这次实践，能够让你对Kodein有更深度的了解：

[MVVM-Rhine:The MVVM Architecture in Android.](https://github.com/qingmei2/MVVM-Rhine)

> MVVM-Rhine，是我目前在尝试探索的mvvm开发架构(Rhine:莱茵河），目前处于摸索和开发期，欢迎参考并提出建议。

### 1.定制Application

在一个Android项目中，很多依赖都需要保持单例，这样能够保证合理的资源规划，比如，Retrofit的实例化，比如Gson对象的实例化，这里我们直接在Application中进行配置：

```kotlin
open class RhineApplication : Application(), KodeinAware {

    override val kodein: Kodein = Kodein.lazy {
       
    }
}
```

`KodeinAware`是一个接口，它意味着，实现该接口的对象都会持有一个Kodein容器：

```Kotlin
interface KodeinAware {
    /**
     * A Kodein Aware class must be within reach of a [Kodein] object.
     */
    val kodein: Kodein
    
    val kodeinContext: KodeinContext<*> get() = AnyKodeinContext

    val kodeinTrigger: KodeinTrigger? get() = null
}
```

`RhineApplication`实现了KodeinAware接口，并实例化了一个`Kodein`容器，接下来我们要做的，就是把对应的依赖装进`Kodein`容器中。

我们定义了一个`httpClientModule`的`顶层属性`以声明Retrofit相关：

```kotlin
const val HTTP_CLIENT_MODULE_TAG = "httpClientModule"

const val TIME_OUT_SECONDS = 20

val httpClientModule = Kodein.Module(HTTP_CLIENT_MODULE_TAG) {
    
    bind<Retrofit.Builder>() with singleton { Retrofit.Builder() }

    bind<OkHttpClient.Builder>() with singleton { OkHttpClient.Builder() }

    bind<Retrofit>() with singleton {
        instance<Retrofit.Builder>()   // 委托给了bind<Retrofit.Builder>()函数
                .baseUrl(APIConstants.BASE_API)
                .client(instance())    // 委托给了 bind<OkHttpClient>() 函数
                .addCallAdapterFactory(RxJava2CallAdapterFactory.create())
                .addConverterFactory(GsonConverterFactory.create())
                .build()
    }

    bind<OkHttpClient>() with singleton {
        instance<OkHttpClient.Builder>()  // 委托给bind<OkHttpClient.Builder>()函数
                .connectTimeout(
                        TIME_OUT_SECONDS.toLong(),
                        TimeUnit.SECONDS)
                .readTimeout(
                        TIME_OUT_SECONDS.toLong(),
                        TimeUnit.SECONDS)
                .addInterceptor(HttpLoggingInterceptor()
                        .setLevel(HttpLoggingInterceptor.Level.BODY))
                .build()
    }

    bind<Gson>() with singleton { Gson() }
}
```
Retrofit声明好了，我们再声明对应的APIService:

```kotlin
const val SERVICE_MODULE_TAG = "serviceModule"

val serviceModule = Kodein.Module(SERVICE_MODULE_TAG) {

    bind<UserService>() with singleton {
        // Retrofit对象的获取已经在httpClientModule中声明好了
        instance<Retrofit>().create(UserService::class.java)
    }

    bind<ServiceManager>() with singleton {
        ServiceManager(instance())  // userService的获取方式已经声明
    }
}

// 目前ServiceManager只有User相关的API接口，可后续慢慢追加
data class ServiceManager(val userService: UserService)
```
当然，这些依赖的绑定都依赖于`项目架构`，比如，我的项目用到了`RxCache`，我也声明了对应的`cacheModule`：

```kotlin
val CACHE_MODULE_TAG = "CacheModule"

val cacheModule = Kodein.Module(CACHE_MODULE_TAG) {

    bind<RxCache>() with singleton {
        RxCache.Builder()
                .persistence(ContextCompat.getExternalCacheDirs(instance())[0], GsonSpeaker())
    }
}
```

这些依赖最终都统一交给`RhineApplication`,在我的项目中，它大概是这样的：

```
open class RhineApplication : Application(), KodeinAware {

    override val kodein: Kodein = Kodein.lazy {
        bind<Context>() with singleton { this@RhineApplication }
        import(androidModule(this@RhineApplication))
        import(androidSupportModule(this@RhineApplication))

        import(serviceModule)
        import(cacheModule)
        import(rxModule)
        import(httpClientModule)
    }
}
```

### 2.定制Activity或者Fragment

全局的依赖交给了`RhineApplication`,如果对于一个Activity，它可能还有其他的依赖需要注入，这意味着，我们需要：

* 1.`RhineApplication`级别的Kodein容器
* 2.`Activity`级别的Kodein容器，它包含仅Activity所需依赖

这很简单，我们只需要`extend` 和 `import`：

```kotlin
class MainActivity : BaseActivity() ，KodeinAware {
    
    private val parentKodein by closestKodein()  // 1

    override val kodein: Kodein by retainedKodein {
        extend(parentKodein, copy = Copy.All)    // 2
        import(mainKodeinModule)     // 3
        bind<MainActivity>() with instance(this@MainActivity)  // 4
    }
    // 注入MainNavigator控制Activity的视图导航
    private val navigator: MainNavigator by instance()   
    // 注入MainViewModel管理业务数据
    private val mainViewModel: MainViewModel by instance()  
}
```

这里的Activity代码仅方便读者理解，实际代码因架构设计有一定偏差：

1.`closestKodein()`函数返回了`相邻上层`的一个Kodein容器，对于`Activity`来说，它返回的是`Application`层级的`Kodein`容器。
2.通过`extend()`函数，我们将Application层级的Kodein容器也放在了`Activity`的kodein容器中，这样`Activity`就能从上层的Kodein容器取出对应依赖（俄罗斯套娃？），比如网络请求的service相关等等。
3.类似`Application`的注入方式一样，我定义了一个`mainKodeinModule`,以存放`MainActiviy`所需依赖的绑定函数,类似dagger中的`@Scope`,其中`scoped(AndroidComponentsWeakScope)`保证了`Activity`级别的`局部单例`，：

```kotlin
val MAIN_MODULE_TAG = "MAIN_MODULE_TAG"

val mainKodeinModule = Kodein.Module(MAIN_MODULE_TAG) {
    // 省略很多代码...
    bind<HomeFragment>() with scoped(AndroidComponentsWeakScope).singleton {
        HomeFragment()
    }

    bind<MainViewModel>() with scoped(AndroidComponentsWeakScope).singleton {
        // 这里需要MainActivity，请参考下文中4的讲解
        instance<MainActivity>().viewModel(MainViewModel::class.java)
    }
}
```
4.正如上文看到的，一些对象的实例化需要`Context`的上下文对象，我们通过`bind<MainActivity>() with instance(this@MainActivity) `完成`MainActivity`的绑定。

## Kodein：小结
---

能看到这里的，基本都是真爱了，实际上，相比于Dagger，Kodein的学习成本更低，代码更简洁，配置更简单。

我不认为这样一篇博客就能 **Kodein从入门到精通**，所谓实践出真知，我更建议您参考实际的项目，去了解它在实际项目中的应用:

https://github.com/qingmei2/MVVM-Rhine

最后列一下相关学习资料（笔者写稿的此时，国内尚未有任何Kodein的中文学习资料，实在遗憾），以供大家参考：

[Kodein官方文档：Getting started with Kodein DI](http://kodein.org/Kodein-DI/?5.2/getting-started#_the_type_erasure_problem)
[Kodein官方文档 for Android：Kodein DI on Android](http://kodein.org/Kodein-DI/?5.2/android)

### 2018/11/30补充

感谢 [o动感超人o](https://www.jianshu.com/u/b433b31eadad) 兄弟花了很久总结的图，将Kodein官方文档晦涩的文字总结了出来，经过他的同意，我把这张图也转载过来，以供大家参考：

![image.png](https://upload-images.jianshu.io/upload_images/7293029-74ef23e2485c53f0.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

原文链接：[《使用Kodein作为Dagger2的升级版替代品》](https://www.jianshu.com/p/a577237538ee)   by  [o动感超人o](https://www.jianshu.com/u/b433b31eadad) 
原图链接：http://naotu.baidu.com/file/7f5ea2ba8e7fa2820973d11b7c66a98a

## 附：笔者Dagger系列相关文章

我个人之前关于Dagger的系列文章：

[ Android 神兵利器Dagger2使用详解（一）基础使用 ](http://www.jianshu.com/p/b40bcd1a9ec9)
[ Android 神兵利器Dagger2使用详解（二）Module&Component源码分析](http://www.jianshu.com/p/30d48ddefd30)
[ Android 神兵利器Dagger2使用详解（三）MVP架构下的使用](http://www.jianshu.com/p/c46acc3f21ab)
[ Android 神兵利器Dagger2使用详解（四）Scope注解的使用及源码分析 ](http://www.jianshu.com/p/caaac320c785)

[ 告别Dagger2模板代码：dagger.android使用详解 ](http://www.jianshu.com/p/917bf39cae0d)
[ 告别Dagger2模板代码：dagger.android原理解析 ](http://www.jianshu.com/p/d4d62945d9c8)

[ 告别Dagger2，在Kotlin项目中使用Kodein进行依赖注入 ](https://www.jianshu.com/p/b0da805f7534)
[ 【译】Android开发从Dagger2迁移至Kodein的感受  ](https://www.jianshu.com/p/e5eef49570b9)

最后这篇是笔者翻译的一篇文章，文中原作者针对Kodein和dagger2进行了一些对比，说实话，笔者对其中一些观点保持自己的意见，如有可能，我将会针对Kodein再写一篇分析类的文章，欢迎关注。
