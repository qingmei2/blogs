# 使用Flutter开发Github客户端及学习历程的小结

本文笔者将尝试分享个人针对`Flutter`的 **学习** 并 **搭建一个Flutter应用** 的过程。

在这一个月学习`Flutter`的过程中，**我不可避免的走了很多弯路**，也许这并非坏事，但是还是希望将这些经历表述出来，有两个目的：

* 1.为自己做一个周期性的总结；
* 2.也希望能给想学习`Flutter`的读者一定实质性的参考。

> 关于笔者总结的`Flutter`入门学习计划，可直接跳转文末的 **Flutter入门学习计划** 小节进行查看。

## 契机

上个月25号，[任玉刚](http://renyugang.io/)老师联系我，问我有没有兴趣翻译一篇`Flutter`的技术博客。

当时我还没有接触`Flutter`，觉得这是一个督促自己学习的机会，就尝试接下了这个任务。截止今天为止（6月25日）刚好一个月，在第一周保证翻译任务 [完成](https://mp.weixin.qq.com/s/fh6lvYONrWmldMEnEdw5yg) 之后，三周之后的今天，我基本实现了自己的另外一个目标——搭建一个 **Github客户端**。

这个项目运行之后，App整体效果是这样的：

![](https://raw.githubusercontent.com/qingmei2/qingmei2-blogs-art/master/android/flutter/github_sample/github_sample.8swu8uj3qvg.png)

我将代码托管在了自己的`Github`上：

[FlutterGitHubApp: Flutter开发的跨平台Github客户端.](https://github.com/qingmei2/FlutterGitHubApp)

因为这是一个入门的项目，所以接下来也会从各方面深入学习`Flutter`，并反过来继续完善和优化它。

## 第一周：初识Flutter

最初学习`Flutter`的方式是通过学习 [wendux](https://github.com/wendux) 老师的 [《Flutter实战》](https://book.flutterchina.club/)。

这是一本非常优秀的中文`Flutter`教程，对个人学习`Flutter`入门有非常大的帮助。

我根据这本小册中的内容完成了第一个 **计数器** 的入门案例，并对最常用的一些控件进行了熟悉和了解：

![](https://raw.githubusercontent.com/qingmei2/qingmei2-blogs-art/master/android/flutter/github_sample/WX20190627-222207%402x.eai12zgahku.png)

正如读者所见，我跟着[《Flutter实战》](https://book.flutterchina.club/) 写了若干的demo代码，遗憾的是，效果并没有想象的那么好，原因也很明显，那就是我还没有完全熟悉`Dart`的语法。

### 磨刀不误砍柴工

学习`Flutter`的最开始，语法并非是最大的阻碍因素，因为对于熟悉`Java`语法的我们来说，`Dart`有很多相似之处，但随着`Flutter`学习的不断深入，有时一些`Dart`独有的语法特性会给我带来困惑，比如 **级联操作符** 、 **var和dynamic关键字的区别** 等等。

正如标题所言，我发现我走入了一个误区，`Dart`语法的学习势在必行。

我学习语法的方式是通过翻阅`Dart`中文网：

[Dart中文社区：http://dart.goodev.org/](http://dart.goodev.org/)

### 第一周的感受

因为是空闲时间学习，因此严格来说学习时间并没有那么多，最初的第一周，笔者花了几个晚上，每天9点下班之后学2～3个小时，熟悉了`Dart`基本的语法和`Flutter`的最常用的基础组件。

严格来说，此时个人依然处于小白水平，勉强摸到了入门的门槛。

私下里也会偷偷吐槽一下`Dart`和`Flutter`，布局写着写着下面连续十数行的 `）,）,）,）;）,）,},),},）;）,）,）,;）,},）;` 真的令人不寒而栗......

## 第二周：状态管理

因为当初接翻译任务时，自己给自己设定了10天的期限（也是为了督促自己学习），因此第二周我需要在前3天内翻译完这篇博客：

* [Widget-Async-Bloc-Service: A Practical Architecture for Flutter Apps](https://medium.com/coding-with-flutter/widget-async-bloc-service-a-practical-architecture-for-flutter-apps-250a28f9251b)

坦白来说，第二周的开始，这篇文章我看不懂，因此我需要学习`Flutter`开发过程中的架构思想。

正所谓窥一斑而知全豹，虽然还没有真正着手`Flutter`项目的开发，但是通过学习`Flutter`的核心——**状态管理**，以及将 **业务逻辑** 和 **UI的渲染** 分开学习，再加上作为一个`Android`开发者，理解这些概念本身就有很大的优势，学习效率自然非常的高。

学习`Flutter`中状态管理的资料，我强烈推荐 [Vadaski](https://juejin.im/user/5b5d45f4e51d453526175c06) 的系列文章。

* [Flutter | 状态管理探索篇——Scoped Model（一）](https://juejin.im/post/5b97fa0d5188255c5546dcf8)
* [Flutter | 状态管理探索篇——Redux（二）](https://juejin.im/post/5ba26c086fb9a05ce57697da)
* [Flutter | 状态管理探索篇——BLoC(三)](https://juejin.im/post/5bb6f344f265da0aa664d68a)
* [Flutter | 状态管理拓展篇——RxDart(四)](https://juejin.im/post/5bcea438e51d4536c65d2232)
* [Flutter | 状态管理指南篇——Provider](https://juejin.im/post/5d00a84fe51d455a2f22023f)

冒昧推荐这几篇关于状态管理的文章，实际上 [Vadaski](https://juejin.im/user/5b5d45f4e51d453526175c06) 老师关于`Flutter`还有很多优秀的博客，这里不一一列举了，有兴趣的朋友可以去拜读一下。

如果读者之前学习或者了解过`Redux`和`ReactiveX`相关的思想，状态管理并不是非常难理解的概念。

熟悉了一系列`Flutter`状态管理的实现方式之后，翻译文章时就顺畅很多了，幸不辱命，最终在第十天的凌晨将文章翻译完毕：

* [Flutter 移动端架构实践：Widget-Async-Bloc-Service](https://mp.weixin.qq.com/s/fh6lvYONrWmldMEnEdw5yg)

完成之后，因为工作和私人的原因，第二周接下来几天就没有什么时间学习`Flutter`了。

### 第二周小结

第二周的学习成果实际上和第一周差不多，因为前三天全神贯注，同时每天晚上多学了一会，再加上吃了之前的老本（之前对于`Redux`的状态管理和`RxJava`有一定的储备），学习效率还是比较高的。

这周的感觉就是，虽然自己没怎么上手项目，但是看了一些文章，对`Flutter`有了一些初步的认识，总结如下：

* 1.因为`Flutter`本身采用的是`React`的思路，和我们认知的 **过程式开发** 是不一样的， **状态管理** 和 **响应式编程** 是非常重要的概念，如果之前有相关的知识储备，这个关键的知识点基本不会有什么难度，只需要关注`API`的使用就好了；当然，没了解过也没关系，本小节上方的几篇关于状态管理优秀的博客，也能够帮助开发者非常快的进入`Flutter`的节奏中去。
* 2.类比是一个非常好的学习方式，对于`Flutter`中的一些概念或者库而言：  
> 2.1 `RxDart`和`Stream`相关的`API`和`RxJava`很相似；  
> 2.2 `Future`相关的API可以参考`Kotlin`的协程，通过同步的方式编写异步的代码；  
> 2.3 `Provider`其实也就是另一种方式的依赖注入.  
> 2.4 `Redux`就是参考前端的`Redux`引进的，没有什么变化......

## 第三周：学习Widget

从结果来看，第三周我走了不少弯路。

第三周的最初，我认为我需要开始深入学习`Flutter`中的`Widget`，因此我选择`fork`了著名的 [flutter-go](https://github.com/alibaba/flutter-go), 并且开始尝试跟着这个项目敲代码。

在敲了几天之后，我发现一个严重的问题，那就是这个学习过程中非常枯燥无聊，知识点之间没有关联性，感觉自己学了一个新的`Widget`，就忘了上一个`Widget`，没坚持多久，我就`hold`不住了......

![](https://raw.githubusercontent.com/qingmei2/qingmei2-blogs-art/master/android/flutter/github_sample/image.ivwp0vhtmig.png)

这也难怪，这个项目本身的目的就是 `常用组件的demo演示与中文文档`, 我一个`Widget`一个`Widget`的用法跟着敲，这给了我一种 **学习碎片没有组织起来** 的感觉，说白了就是不成系统，效果并不明显。

因此我将 [flutter-go](https://github.com/alibaba/flutter-go) 这个项目的定位变成了 **工具书** ，接下来的学习过程中，每当我对一个`Widget`的使用有了疑问，就随手打开这个APP进行查阅这个`Widget`的用法，效果还不错。

## 第四周：在实战中学习

第四周我选择了实战开发，了解我的朋友应该知道，我曾经通过不同的开发模式（`MVVM`和`MVI`）开发过两次`Github`的客户端，这次我也不例外。

选择以`Github`客户端作为实战的练手项目还有一个原因，那就是 [恋猫de小郭](https://juejin.im/user/582aca2ba22b9d006b59ae68) 老师已经开源了一个更强大的`Github`客户端可以作为参考：

[GSYGithubAppFlutter: 超完整的Flutter项目](https://github.com/CarGuo/GSYGithubAppFlutter)

同时，[恋猫de小郭](https://juejin.im/user/582aca2ba22b9d006b59ae68) 老师也有非常优秀的`Flutter`系列博客，因为该系列文章太多了，就不一一列出了，强烈建议收藏阅读。

所谓前人栽树后人乘凉，[GSYGithubAppFlutter](https://github.com/CarGuo/GSYGithubAppFlutter) 确实在我实践过程中提供了很大的帮助，同时，因为第四周工作阶段性告一段落，我有更多时间去学习`Flutter`，因此很快就把一个简单的`Github`客户端敲了出来：

https://github.com/qingmei2/FlutterGitHubApp

## 阶段性总结

在一个月的学习过程中，我学习到了很多东西，也感觉很多地方需要慢慢改进，也感觉到有很多知识点需要去补。

但是令我振奋的一点是，我成功从舒适区跳了出来，并且度过了学习新知识过程中最痛苦的一段时间（畏难情绪+新领域的陌生感）；

现在面对诸如 **`Kotlin`和`Flutter`我学哪个比较好?** 的问题，我也可以这样回答了：

> **小孩子才做选择，成年人当然是全都要啦！**

最后，衷心感谢文中提到的各位老师对个人的帮助，其实在学习过程中，我还参考了更多`Flutter`先驱者们优秀的博客和代码，实在难以一一列举，在此深表感谢。

### Flutter入门学习计划？

如何入门`Flutter`？ 以个人经验来看，入门学习`Flutter`可以参考下面步骤：

* 1.通过[《Flutter实战》](https://book.flutterchina.club/)电子书完成一个简单的计时器示例；
* 2.通过 [Dart中文社区](http://dart.goodev.org/) 学习语法；
* 3.继续学习[《Flutter实战》](https://book.flutterchina.club/)，了解`Flutter`基本概念；
* 4.下载 [`flutter-go`](https://github.com/alibaba/flutter-go),将`App`下载到手机中作为工具书随时随地查阅；
* 5.1.学习一些优秀的`Flutter`博客系列，比如上文中 [Vadaski](https://juejin.im/user/5b5d45f4e51d453526175c06) 和 [恋猫de小郭](https://juejin.im/user/582aca2ba22b9d006b59ae68) 两位老师的文章；
* 5.2 同时，下载优秀的`Flutter`开源项目学习源码；
* 6. 选择一个感兴趣的项目或者方向进行实战练习。

> 这个学习计划 **一定是有改进空间** 的，也诚挚的希望您能在评论区留下宝贵的想法和建议，这也能够为读者提供更多参考性的建议，感谢！

---

## 关于我

Hello，我是[却把清梅嗅](https://github.com/qingmei2)，如果您觉得文章对您有价值，欢迎 ❤️，也欢迎关注我的[博客](https://www.jianshu.com/u/df76f81fe3ff)或者[Github](https://github.com/qingmei2)。

如果您觉得文章还差了那么点东西，也请通过**关注**督促我写出更好的文章——万一哪天我进步了呢？

* [我的Android学习体系](https://github.com/qingmei2/android-programming-profile)
* [关于文章纠错](https://github.com/qingmei2/Programming-life/blob/master/error_collection.md)
* [关于知识付费](https://github.com/qingmei2/Programming-life/blob/master/appreciation.md)
