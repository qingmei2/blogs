
![大会现场](https://upload-images.jianshu.io/upload_images/7293029-46d37dadb753c919.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

> 本文由 [玉刚说写作平台](http://renyugang.io/%E5%85%B3%E4%BA%8E%E6%88%91/) 提供写作赞助，赞助金额：**300元**  
原作者：[却把清梅嗅](https://blog.csdn.net/mq2553299)  
原文地址：[https://mp.weixin.qq.com/s/h0HUyrpbDtbEeiY2Z3-POQ](https://mp.weixin.qq.com/s/h0HUyrpbDtbEeiY2Z3-POQ)
版权声明：本文版权归微信公众号 **玉刚说** 所有，未经许可，不得以任何形式转载！  

## 前言

美国当地时间5月8日，2018年 Google I/O开发者大会在美国加州山景城拉开帷幕。

这是全球5月份最盛名的一次开发者大会，即使你不是一名专业的技术人员，你也可以从中获取不少前沿性的内容——当然，它更是程序开发者们的特殊节日，在I/O大会开始之前，相关网站就已经浓墨重彩开始了宣传（作为Android开发者，笔者惊奇地发现[Android Developers](https://developer.android.com)官方网站也迎来了全新的改变）。

在本届大会上，AI(人工智能)成为贯穿全场的主题。谷歌不仅发布了新一代为机器学习定制的芯片TPU（张量处理器）、结合了AI技术的 **Android P** ，还升级了不少AI应用 ——不难发现，Google 把开发重心更多放到了AI上面，除此之外，**移动端技术**和 **前端技术** 的发展也在Google的重点关注之中，这之后，谷歌还谈到了在自动驾驶领域和TPU芯片研发的新进展。

作为一个Android开发者，我们应该做到的是关注最新的技术动态和风向，我尝试花了一些时间总结了本次大会的 **干货列表**，并简单做了一下总结，希望对大家能有所帮助。

本文的目录如下：

* AI的演示——Google Assistant 
* **Android P**
* **Android Jetpack**
* **Kotlin的上位**
* Android Wear
* Android Things
* ARCore
* Instant app
* 总结

## 一.AI的演示——Google Assistant 

如果说AI是贯穿全场的主题，那么 **Google Assistant(谷歌助手)** 就是GoogleCEO **Sundar Pichai** 本次大会握在手中的杀手锏之一。

在此之前，各大厂商都推出过自己的 **AI语音助手**，以笔者接触使用过的为例，Apple的 **[Siri](https://baike.baidu.com/item/siri/32248?fr=aladdin)**，Windows的 **[Cortana](https://baike.baidu.com/item/Cortana/13209604?fr=aladdin)**，以及小米的**[小爱同学](https://baike.baidu.com/item/%E5%B0%8F%E7%88%B1%E5%90%8C%E5%AD%A6/22047751?fr=aladdin)**。这些 **AI助手** 都给我日常生活中带来了不少便利，但是不可避免的，它们都有相同的硬伤，那就是**对话生硬、无法根据情境进行复杂会话**。

而本次大会上，Google Assistant向全场展示了AI技术的迅猛发展：用户对Google Assistant说想剪头发，GoogleAssistant立即拨通了理发店的电话，并进行预约剪发。

在整个 **剪发预约** 的过程中，**Google Assistant**展示出了一场流利的交流：在理发店说出“我需要查查Jim老师的档期，稍等”时，**Google Assistant**出乎意料的一句“Mm-hmm(嗯哼)”惊艳全场—— 对于AI来讲，根据不同情境发出对应的 **语气词**，这种 **自然的方式** 向对方示意，使会话变得摆脱了生硬感。

![](https://upload-images.jianshu.io/upload_images/7293029-f629f7bf7ba004d3.gif?imageMogr2/auto-orient/strip)

通过见证Pixel XL升级到 **Android P**以体验**GoogleAssistant**后，得出的结论是，目前**GoogleAssistant**还不支持中文的交流，同时，**GoogleAssistant**能否在所有场景中具有普适性，目前依然无法得知。

## 二.Android P

最新的Android P系统中，更加注重人工智能方面的探索，也推出了众多的新功能，比如下面这些：

* Google Assistant
* 推出Android设备的 **刘海全面屏**
* 全新的音量调节显示效果
* 适配全面屏，将Bottom传统的Menu,Home,Back键替换成了**药丸**按键
* 推出 **省电模式**（Adaptive Battery）
* 更多 **AI** 相关功能的支持，比如电量优化

### 开始准备适配刘海屏？

![刘海屏](https://upload-images.jianshu.io/upload_images/7293029-df002fc903d1929e.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

对于Android开发者来说，**刘海屏** 的适配似乎已经不可避免，所幸的是，因为 **刘海屏** 只涉及到了状态栏，因此，**适配方案** 还是比较清晰有条理的：

* 对于有状态栏的页面，不会受到刘海屏特性的影响；

* 全屏显示的页面，系统刘海屏方案会对应用界面做下移处理，避开刘海区显示；

* 已经适配Android P应用的全屏页面可以通过谷歌提供的适配方案使用刘海区，真正做到全屏显示。

更多详细，请参考[这篇文章](https://my.oschina.net/u/3787471/blog/1788756)。

## 三.Android Jetpack

在本次开发者大会上，Google发布了**Android Jetpack**，这是我们的新一代 **组件、工具和架构指导** ，旨在加快开发者的 Android 应用开发速度。 

这对 **Android原生开发** 来讲，无疑是个 **重磅炸弹**，众所周知，2017年的GoogleIO大会上，官方为Android开发者推出了 一系列的 **架构组件**，比如 **Lifecycle**，**LiveData**，**Room**，**ViewModel**等，旨在推出 **官方认证** 的 **Android架构** 方案。

而本次IO大会上，**Android Jetpack**的推出，从 **架构（Architecture）**，**UI**，**基础（Foundation）**及**行为（Behavior）**四个方面，推出了一套 **官方认证** 的 **开发体系**。

让我们通过一张图了解 **Android Jetpack** 的详细组成：

![Android Jetpack](https://upload-images.jianshu.io/upload_images/7293029-061f5d645fd0c7a4.jpeg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


本次的 **Android Jetpack** 推出了五个新组件，它们分别是：

* Navigation
* Paging
* WorkManager 
* Slices
* Android KTX

### Navigation——导航组件

Navigation的作用正如其名，为页面的跳转做 **导航** ，更准确的来讲，Navigation是 **单Activity多Fragment** 开发模式下的页面跳转的 **导航组件**。

说到Fragment的栈管理，一直都是很令人头痛的问题，当然，Github上也有一些很优秀的三方库可以解决这个问题，比如著名的 **[Fragmentation](https://github.com/YoKeyword/Fragmentation)** 。

官方的介绍文档中，Navigation具有以下强大的导航功能：

> 利用导航组件对 Fragment 的原生支持,可以获得架构组件的所有好处（例如生命周期和 ViewModel），同时让此组件为您处理 FragmentTransaction 的复杂性。此外，Navigation组件还可以让您声明我们为您处理的转场。它可以自动构建正确的“向上”和“返回”行为，包含对深层链接的完整支持，并提供了帮助程序，用于将导航关联到合适的 UI 小部件，例如抽屉式导航栏和底部导航。

不难看出，Navigation作为Fragment栈管理的 **官方解决方案**，还是很强大的，和其它三方库相比，它拥有的优势有：

* 专业的 **开发/维护** 和 **测试** 团队，保证代码的质量及稳定性；
* AndroidStudio **IDE专属支持**，包括可视化的编辑界面，和通过鼠标拖拽对Fragment的导航管理功能；
* 对ViewModel，Lifecycle等 **官方架构组件** 的支持；
* 官方针对**迁移至Navigation** 的流程提供了详细的文档；
* 更多Android开发者会使用它，您可以在网上获取非常详尽的学习资料；
* Google爸爸官方出品，无脑支持。

抛开比较性的话题不谈（StoryBoard VS Navigation?），笔者认为这个库还是有学习的必要性的——当然，它的优先级不高，因为不是任何项目都是 **单Activity多Fragment**的架构，但是，作为本次 **Android Jetpack**力推的组件之一，我们有必要在新的项目中或者重构中，在对Fragment的设计时，**优先考虑** 使用 **Navigation**。

关于Navigation更详细的内容，可以参考笔者的**《Android Jetpack Navigation：大巧不工的Fragment管理框架》**这篇文章：
https://www.jianshu.com/p/ad040aab0e66

### Paging——分页组件

对于Android开发中的 **数据列表展示** ，很多时候我们需要对数据进行 **分页加载**，这时候我们可以考虑 使用 **Paging**。

让我们来看看官方对于 **Paging** 的相关介绍：

> 应用中呈现的数据可能非常大，这就导致加载的开销比较大，因此，避免一次下载、创建或呈现过多数据就显得非常重要。 [分页组件 ](https://d.android.com/arch/paging)让您可以轻松加载和呈现大型数据集，同时在您的 RecyclerView 中进行 **快速、无限滚动**。它可以从本地存储和/或网络加载 **分页数据**，并让您能够定义内容的加载方式。此组件原生支持 Room、LiveData 和 RxJava。

Android系统对大量数据的列表展示，RecyclerView 绝对是不二之选，但是关于数据的分页展示，目前有很多种解决方案。而本次IO大会，谷歌推出了这款 **Paging**，它和其它三方库相比，其优势有：

* 专业的 开发/维护 和 测试 团队，保证代码的质量及稳定性；
* 原生支持Room、LiveData 和 **RxJava**；
* 官方针对分页库 **迁移至Paging** 的流程提供了详细的文档；
* 更多Android开发者会使用它，您可以在网上获取非常详尽的学习资料；
* Google爸爸官方出品，无脑支持。

令人振奋的是，**Paging** 不但提供了对 **LiveData** 以及 **Room**的原生支持，更是提供了对 **RxJava** 的官方原生支持，这也侧面反映了 Google 对 **RxJava** 的认可肯定。

作为 **业务层** 和 **UI层** 的链接， **Paging** 分页库的适用场景是非常多的，在新项目的架构或者项目重构中，它是一个优先级比较高的选择—— 当然，如果您对于 **RxJava** 情有独钟，**Paging** 这个令人振奋的额外支持一定不会让您失望。

关于 **Paging** 更详细的内容，请参考：https://developer.android.com/topic/libraries/architecture/paging/

### WorkManager

WorkManager是一个很新颖的库，它的作用一句话概述就是:

> 管理一些要在后台工作的任务, ——即使应用没启动也能保证任务能被执行。

和 **RxJava** 以及 **AsyncTask** 不同的是，前者帮助你在应用中开后台线程干活, 但是应用一被杀或被关闭, 这些工具就干不了活了。

而WorkManager不是, 它在应用被杀, 甚至设备重启后仍能保证你安排给他的任务能得到执行。

它的特点在于：

* **易于安排**：您可以在 **特定条件** 下启动任务，同时，任务之间可以相互链接，这意味着，你可以将任务 **串行** 或者 **并行** 执行。
* **易于取消**：您拥有对任务执行的控制权，通过API您可以轻松取消计划任务。
* **易于查询**：您可以将任务进度展示在各种各样的UI上。
* **支持所有Android版本！**：就像描述的一样，各个Android版本下，WorkManager的API都是一致的。

听起来很棒，看到这些特性时，我第一感觉就是**APP定时唤醒**的功能——流氓软件再次迎来了春天？

WorkManager 的上手难度并不大，关于 **WorkManager** 更详细的内容，这里有一篇文章也许能有所参考：https://juejin.im/post/5af4aa91f265da0b8d41f714

### Slices —— 切片

> [Slices](https://developer.android.com/guide/slices/) 提供了一系列 UI 模板，帮助开发者在应用中呈现丰富的动态交互式内容，支持**所有 Android 系统以及提供谷歌服务的平台** 。

Android Jetpack内置了对Slices的支持，并可以最低支持Android 4.4——大约占所有Android用户的95％。

这是一个很有趣的特性，在公司内部的交流会的演示中，通过Google 搜索引擎进行关键字检索时，如果APP配置了Slices，可以在Google搜索结果中显示来自APP的丰富，动态和交互式内容。

遗憾的是，该行为的前提是—— **Google搜索**，是的，对于国内的Android厂商来讲，Slices很有可能会被阉割。

当然，在实际结果没有出来之前，Android开发者在没有需求的情况下，可以不优先对Slices进行深入学习，保持简单了解即可。

## 四.Kotlin的上位 & Android KTX

![](https://upload-images.jianshu.io/upload_images/7293029-69da8c7c81cd8c9f.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

如果说，去年的Google IO大会掀起了一波 Android开发者们对于 **Kotlin** 语言学习的热潮的话，那么今年的 IO大会，Google向我们展示出了 **Kotlin** 排山倒海般势不可挡的推动。

在今年的 Google I/O 大会上，关于 Kotlin，Google 只说了 **只言片语**：

> 在过去一年里，Play Store 中用 Kotlin 开发的应用在去年增至 6 倍，在高级开发者中有 35% 的人选择使用 Kotlin 进行开发，95% 的开发者表示很喜欢用 Kotlin 进行 Android 的开发，而且这个数字正在逐月递增。

同时，Google 会继续改善 Kotlin 在支持库、工具、运行时 (runtime)、文档以及培训中的开发体验。Google 在今年2月发布的 **Android KTX**，也会包含在上面提到的 **Android Jetpack** 中，力图优化 Kotlin 开发者体验；同时继续改善 Android Studio、Lint 支持以及 R8 优化中的工具；而且对 Android P 中的运行时 (Android Runtime) 进行微调，以此加快 Kotlin 编写的应用的运行时间。

这之后，Google就没有再宣布关于 Kotlin 的重大消息或规划了。但是细心的同学可以发现，Google 对于 Kotlin 的 **偏向** 已经 **十分明显**：无论是今年IO大会的 **代码展示**，还是官方文档上的 **代码示例**，亦或是Google最新 **开源Demo的源码**，都是以 Kotlin语言为主。

时至今日，我认识的不少小伙伴，对于Kotlin语言都仍旧保持观望的态度，理由大概都是相似的：

* 我 **简单学习** 了Kotlin，但是我并不太会用，因为我没有机会在项目中使用它；
* Kotlin相比于Java, 依然还是太年轻，对于成熟的项目来讲，Java依然是短时间不可替代的

类似种种，事实证明，一年过去了，理由还是一年前的理由，但是 **Kotlin** 发展的势头已经 **突飞猛进**，如果说，去年的此时，**熟练使用Kotlin语言进行开发** 是个人简历中的一大亮点，也许不到明年，这个技能点就将是各中大型公司招聘时 **最基本的一个门槛** 了。 

这也许有点危言耸听，但事实上，作为一个开发人员，对 **新技术** 的持续关注以及 **个人学习能力** 都是各互联网公司所看重的——对于Android开发来讲，Kotlin的使用就是这样的一个典型。

## 五.AndroidWear

本次大会上，Google发布了新的 Wear OS的 **开发者预览版**，为 Wear OS 带来 Android P 平台的心功能。对 Wear OS 中国版的开发者，Google特别提供了一个更新的模拟器镜像以及可以刷到华为第二代智能运动手表（蓝牙版）上的系统镜像。

本次关于AndroidWear的更新主要包括以下几点：

* **全新省电模式**
* **更多功耗优化**，包括蓝牙连接断开时关闭 Wi-Fi 以及 后台活动与前台服务限制
* **通知智能回复**

我是Google的粉丝，但是说实在的，Android端的智能手表，我实在无法昧着良心为它洗地 [捂脸]。

笔者负责的项目有 AndroidWear 的相关项目，但该 Wear 项目主要功能也是通过 **蓝牙** 连接以及将 **数据传递** 给手机端应用，更多功能，AndroidWear还不能够完全的胜任——最主要的是，开发调试时，AndroidWear使用起来的 **繁琐** 有时真的令人发狂。

无论是AppleWatch，还是Android Wear,所能提供的生活帮助都是有限的，直白点说吧， **我没钱买这么一款性价比不高的产品**。

性价比不高并不代表 **智能手表** 不好，但一定是导致其 **销售市场表现不佳**的最主要的原因之一，屏幕比较小 ，加上 **科技发展** 所能带来的 **智能化** 还不足，Android Wear的这次更新只能沦为配角。

## 六.Android Things

![image](http://upload-images.jianshu.io/upload_images/7293029-fa292eaf0af028c8.jpeg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

本次大会，Google正式发布了 **AndroidThings** 的1.0版本。Android Things 是 Google 的托管操作系统，可以让您大规模构建和维护物联网设备。运用 Google 的后端基础设施，我们通过认证硬件、丰富的开发者 API 和安全的托管软件更新提供了一个可靠平台，它可以完成众多繁重的工作，让您将精力集中在构建产品上。

一句话说，Android Things就是让开发者 **可以使用Android开发工具开发嵌入式设备**—— 只要你会开发APP，你就能开发智能设备。

如今的Android开发，纯原生软件开发的竞争力逐渐变弱，而无论是 **Linux**，还是**RN**，亦或是**人工智能**，这些方向对于单一的技术栈都是一个强力的补充。

而闲暇时间，通过 **AndroidThings** 的机会，尝试横向学习一些东西，比如Linux，顺便开发一些有趣的小零件，还是比较有趣的。

笔者对此涉猎不多（买的Raspberry Pi 如今还在家里吃灰w），这里
推荐一个收藏的AndroidThings的博主，有很多AndroidThings的文章可以学习一下：

https://www.jianshu.com/u/c08ebd48644f

## 七.ARCore

Google 今年发布的ARCore，是基于Android 平台的增强现实软件工具开发包，为 Android 设备带来 AR（增强现实）体验。使用 ARCore 构建的应用程序可以识别用户所处的环境，并将物体和信息呈现其中，为用户带来很多既有用又充满乐趣的体验。

目前部分应用了ARCore 的APP已在小米应用商店上线，可在搭载ARCore 的小米MIX 2S上运行。

比如，您可以通过 **居然设计家DIY**，拍一张房间的照片，您就可以给墙面刷涂料、贴壁纸，给地面换地板、瓷砖，给房间添置真实的家具。进入AR模式，您更可以直接体验家具放在家中的样子……

![image](http://upload-images.jianshu.io/upload_images/7293029-48df160b1504e885.gif?imageMogr2/auto-orient/strip)

通过AR，似乎已经可以在家模拟家具摆放，并且下单购买了，看起来Amazing，实际上使用起来还是有一定的误差的。

经过这些年的发展，AR的技术 **逐渐成熟**，并且能够有各自的适用场景，但是目前它依然是一门比较 **小众** 的技术，并且对于开发团队有着一定的要求。

如果您对于ARCore比较感兴趣，您可以参考GoogleARCore的官方网站：
https://developers.google.com/ar/

## 八.Instant app

Instant app 是谷歌推出的类似于 **微信小程序**（或者说小程序类似于instant app）的一项技术，用户无须安装应用，用完就走，同时兼备h5的便捷和原生应用的优质体验。

听完上面的简介，除了 **微信小程序**，我们不禁联想到了不久前 九大手机厂商推出的 **快应用**。相似的点有，**使用前端技术栈开发，原生渲染，同时具备H5页面和原生应用的双重优点**。

InstantApp是Google推出的一个尝试，对于国内的Android开发环境来说，**影响或许不大**，原因和Slices相似，InstantApp的实现也是需要用户手机中安装 **Google play**。

如何快速创建一个InstantApp，这里有一篇文章《instant app入门和开发指南》，讲解的比较详细：
http://renyugang.io/2018/05/24/instant-app%E5%85%A5%E9%97%A8%E5%92%8C%E5%BC%80%E5%8F%91%E6%8C%87%E5%8D%97/

## 总结

本次IO大会上，还有更多的新鲜好玩的技术被提出，比如 MachineLearning的 **TensorFlow** ,升级后的 **MaterialDesign** ,跨平台开发框架 **Flutter**，web端的**Progressive Web App（PWA）**，以及 **VR** 等等。

站在Android开发人员的视角而言，我们可以看到，技术正在不断朝着 **前端** 方向以及 **人工智能** 的大方向发展着，这意味着，在不久的将来，掌握着 **前端开发能力** 的Android工程师将会比 **原生Android开发工程师** 更受青睐，我们在不断提高Android技术栈的同时，横向的技术拓展亦是值得去 **自我投资** , 而前端技术和移动端技术贴合也逐渐紧密，是一个非常好的选择。

最后为大家留下Google 2018 I/O大会上的 **所有演讲视频** 的油管地址，但是您需要 **自备梯子** 才能观看它们：

[Google I/O 2018 - All Sessions](https://www.youtube.com/watch?v=ogfYd705cRs&list=PLOU2XLYxmsIInFRc3M44HUTQc3b_YJ4-Y)

## 感谢

在本文的写作过程中，笔者参考了以下网站的文献及资料，在此深表感谢：

* **GoogleI/O大会官方网站**:  
https://events.google.com/io  
*  **Android Developers官方网站**:  
https://developer.android.com
* **Google开发者中文博客**：  
http://developers.googleblog.cn
* 谷歌2018开发者大会：AI“贯穿一切”,在这里读懂谷歌和人类未来:
http://www.sohu.com/a/230944847_117373
* 最详细的Android P版本刘海屏适配指南来了
https://my.oschina.net/u/3787471/blog/1788756
* instant app入门和开发指南  
http://renyugang.io/2018/05/24/instant-app%E5%85%A5%E9%97%A8%E5%92%8C%E5%BC%80%E5%8F%91%E6%8C%87%E5%8D%97/
