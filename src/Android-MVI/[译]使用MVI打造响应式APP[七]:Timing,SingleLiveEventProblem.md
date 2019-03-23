# [译]使用MVI打造响应式APP(七):定时器(SingleLiveEvent问题)

> 原文：[REACTIVE APPS WITH MODEL-VIEW-INTENT - PART7 - TIMING (SINGLELIVEEVENT PROBLEM)](http://hannesdorfmann.com/android/mosby3-mvi-7)  
作者：[Hannes Dorfmann](http://hannesdorfmann.com)  
译者：[却把清梅嗅](https://github.com/qingmei2)  

在之前的文章中，我们探讨了正确状态管理的重要性，以及我为什么认为使用类似 [Github上Google架构组件的这个repo](https://github.com/googlesamples/android-architecture-components/issues/63) 中的 **SingleLiveEvent** 并不是一个好主意——这种解决方案只是隐藏了真正的潜在问题，那就是**状态管理**。本文我将会阐述`SingleLiveEvent`声称能解决的问题，在`Model-View-Intent`中如何通过状态管理正确地解决。

> 译者注：关于`SingleLiveEvent`的这个[issue](https://github.com/googlesamples/android-architecture-components/issues/63) 从17年讨论到19年至今还未close，各方大佬（还有google的巨佬）针对`SingleLiveEvent`进行了激烈的讨论，堪称Android论坛的一场神仙大战，非常值得一看。

当`error`发生时，**Snackbar** 将被展示—— 这是一个常见的场景可以用来描述这个问题。`Snackbar`并非一直展示，当错误信息被展示几秒钟之后它将消失，问题在于，我们如何模拟这种错误状态并控制`Snackbar`的消失呢？

通过下面的视频，你就能了解我在说什么：

![](pic1)

这个示例显示了如何从 **CountriesRepository** 中加载国家的列表，当我们点击某个国家的条目时，程序将会跳转到第二个界面去展示“详情”（仅仅是国家的名称）。当我们返回到国家列表的界面时，我们希望和之前点击国家条目时的状态一致。

目前为止一切正常，但是当我们执行下拉刷新操作时，一个异常出现了，并通过在屏幕中展示一个`Snackbar`来展示错误信息。正如您在视频中看到的，当我们再次从国家详情返回国家列表时，`Snackbar`和相关的错误信息再次展示了出来，这和用户预期的并不一致，不是吗？
