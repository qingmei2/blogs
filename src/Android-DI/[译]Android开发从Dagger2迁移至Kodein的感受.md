 > 原文：[From Dagger2 to Kodein: A small experiment](https://medium.com/@AllanHasegawa/from-dagger2-to-kodein-a-small-experiment-9800f8959eb4)
作者：[Allan Yoshio Hasegawa](https://medium.com/@AllanHasegawa "Go to the profile of Allan Yoshio Hasegawa")
译者：[却把清梅嗅](https://github.com/qingmei2)

## 译者说
---
我是[却把清梅嗅](https://github.com/qingmei2)，一个普通的Android开发者。去年，我写了一系列关于Android开发者依赖注入框架 **Dagger2** 及 **dagger.android** 的博客：

> [Android 神兵利器Dagger2使用详解（一）基础使用](http://blog.csdn.net/mq2553299/article/details/73065745)
> [Android 神兵利器Dagger2使用详解（二）Module&Component源码分析](http://blog.csdn.net/mq2553299/article/details/73136396)
> [Android 神兵利器Dagger2使用详解（三）MVP下的使用](http://blog.csdn.net/mq2553299/article/details/73251405)
> [Android 神兵利器Dagger2使用详解（四）Scope注解使用及源码分析](http://blog.csdn.net/mq2553299/article/details/73414710)
> [告别Dagger2模板代码：dagger.android使用详解](http://blog.csdn.net/mq2553299/article/details/77485800)
> [告别Dagger2模板代码：dagger.android原理分析](http://blog.csdn.net/mq2553299/article/details/77725912)

最近个人在尝试构建 **Kotlin版本** 的Android MVVM开发框架，在依赖注入框架的选型上，我最终选择了 **[Kodein](https://github.com/Kodein-Framework/Kodein-DI)** 。

这是一个非常轻量级的DI框架，相比于配置繁琐的Dagger（繁琐的配置也是导致Dagger学习成本一直居高不下的原因！），它的配置过程更清晰且简单，并且，这个库的源码也是 **Kotlin** 的。

有同学说，虽然 **Dagger2** 配置很繁琐，但 **dagger.android** 已经大大减少了模板代码的配置。确实如此，但它终究是通过编译器自动生成 **Java** 代码的库，我已经厌倦使用它了......请不要再劝我将它掺入 **Kotlin** 的项目中了。

扯了这么多，请阅读正文吧, 作者简单阐述了 **[Kodein](https://github.com/Kodein-Framework/Kodein-DI)** 的使用感受，通过和 **Dagger2** 对比，阐述库本身的优缺点，希望能给同行一些参考。

> 不久之后，我会专门写一篇文章剖析**[Kodein](https://github.com/Kodein-Framework/Kodein-DI)**的 **入门教程** 和 **项目中的应用** ,敬请期待。

### 2018.9追加

`Kodein`的中文博客详解已更新：

> **[告别Dagger2，Android的Kotlin项目中使用Kodein进行依赖注入](https://www.jianshu.com/p/b0da805f7534)**

# 正文
---
## 从Dagger2迁移至Kodein的感受

有些时候，Dagger2可能会有点过于复杂。 例如，当每个 Activity 都有一个 Scope（作用域） 时，每个屏幕都必须实现一个Scope，一个Module和一个Component。

对于Kotlin的开发者来讲，Kodein将是你的救星。

### 1.Kodein并不是一个依赖注入（DI）框架

虽然 **Kodein** 全名为 **KO**tlin **DE**pendency **IN**jection，但Kodein并不是一个真正的依赖注入框架。 他们的官方文档将其称作 **依赖检索容器** 。

Kodein使用容器来传递依赖关系，这与Dagger2有什么不同？ 让我们看一下从文档的Kodein示例代码：

```kotlin
val kodein = Kodein {
    bind<Dice>() with provider { RandomDice(0, 5) }
    bind<DataSource>() with singleton { SqliteDS.open("path/to/file") }
}
class Controller(private val kodein: Kodein) {
    private val ds: DataSource = kodein.instance()
}
```


第一个表达式创建了一个容器，然后将容器传递给Controller。这之后，将检索依赖项的工作 **委托** 给了Controller。

而使用Dagger2，该示例将如下所示：
  
![](https://upload-images.jianshu.io/upload_images/7293029-4620c2b48669eb72.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

Dagger2中没有 **容器** 的概念。 依赖关系通过构造函数，属性或方法注入。 Controller不知道dataSource的来源——这是Dagger2和Kodein之间的根本区别。

## 2.Kodein中可以使用“非常简单易读的声明性DSL”

官方文档是这样描述的， 上面的代码示例也正是这样做的。 它的好处是使代码更加紧凑，嗯，仅此而已 :)

## 3.使用了Kodein的开源项目？

网上有大量资料可供学习Dagger2：教程，博客文章，开源项目; 它们非常有用。

对于Kodein？ 并不多😔我仍然有很多关于如何正确使用Kodein的疑问。

## 4 Kodein有很多方式可以实现依赖注入

以上文的代码为例，如下所示：

```kotlin
val kodein = Kodein { ... }
class Controller(val dataSource: DataSource)

val controller = kodein.newInstance { Controller(instance()) }
```

现在Controller不依赖于一个容器，这更类似Dagger2通过构造函数传递依赖关系的的实现方式。

或者我们可以这样做：

```kotlin
val kodein = Kodein { ... }
class Controller {
    private val injector = KodeinInjector()
    private val dataSource: DataSource by injector.instance()
    
    init {
        injector.inject(kodein)
        // use dataSource
    }
}
```

这类似于Dagger2注入属性的实现方式。

而且，如果你“勇敢”地在你的应用程序中滥用反射，你甚至可以将JSR330与Kodein一起使用。

关键是，Kodein提供了许多**注入**依赖关系的方法， 您不必总是将**容器**传递给对象。

## 5.你的类可能依赖于Kodein
在使用Dagger2的项目中，您的类将取决于JSR330标准的注解（译者注，@Inject等注解）。 这些类并不依赖Dagger2。

在使用Kodein的项目中，您的类可能需要依赖于Kodein; 这一切都取决于使用哪种实现方式。 在第一个示例中，Controller依赖于Kodein容器。 在**注入器**示例中，Controller依赖于KodeinInjector。

## 6.Kodein在运行时判断依赖关系
如果您忘记将实例绑定到容器，则只会在运行时收到错误消息。 而使用Dagger2，您在编译时会收到错误。

## 7.Kodein容器太通用了
在Dagger2中，我们在定义好的方法中声明模块和组件。 它们可以让Contract了解到谁被谁依赖，依赖被注入在哪里。

但是，Kodein容器没有提供或保存的任何类型信息。 这意味着，对于类型系统，这两个容器是相同的：

```kotlin
val kodein = Kodein {
        bind<Dice> with provider { RandomDice(0, 5) }
}
val foo = Kodein {
        bind<UserRepository> with singleton { UserRepo() }
}
```

如果你很幸运，不小心将foo而不是kodein传递给一个对象。 该项目将成功编译，但它将在运行时发生崩溃。

## 还有？

我喜欢在这个小项目中使用Kodein。 我认为它的DSL非常简单，功能强大且简洁。 但我确实遇到了只在运行时捕获的问题。

我可能会在小项目上再次使用Kodein。 对于更大的项目，我可能还是需要保持观望的态度。
