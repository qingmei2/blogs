# [译]使用MVI打造响应式APP(四):UI组件的独立性

> 原文：[REACTIVE APPS WITH MODEL-VIEW-INTENT - PART4 - INDEPENDENT UI COMPONENTS](http://hannesdorfmann.com/android/mosby3-mvi-4)  
作者：[Hannes Dorfmann](http://hannesdorfmann.com)  
译者：[却把清梅嗅](https://github.com/qingmei2)  

这篇博客中，我们将针对如何 **如何构建独立组件** 进行探讨，我将阐述为什么在我看来 **父子关系会导致坏味道的代码**，以及为何这种关系是没有意义的。

有这样一个问题时不时涌现在我的脑海中—— `MVI`、`MVP`、`MVVM`这些架构设计模式中，多个`Presenter`（或者`ViewModel`）彼此之间是如何进行通讯的？更直白点说吧，`Child-Presenter`是如何与`Parent-Presenter`通讯的？

![](http://hannesdorfmann.com/images/mvi-mosby3/wtf.jpg)

对我来说，这种 **父子关系** 会产生坏味道的代码，因为这直接 **导致了父子层级之间的耦合，使得代码难以阅读和维护**。

这种情况下，**需求的更改会影响很多的组件**（对于大型系统来说，这种情况下实现需求的变动简直难如登天）；并非仅此而已，同时，这也 **引入了难以预测的共享的状态**，其导致的问题甚至难以重现和调试。

其实这也没那么不堪，但我实在不理解为何信息必须从`Presenter A`流向`Presenter B`呢？或者`Presenter`如何与另一个`Presenter`进行通信？

**根本没必要！** 什么情况下`Presenter`才会需要和`Presenter`进行直接的通讯，是什么事件发生了吗？`Presenter`根本不需要和其它的`Presenter`直接通讯，它们都观察了同一个`Model`(或者说是业务逻辑的相同部分),这就是它们如何获得变化的通知：通过底层。

![](http://hannesdorfmann.com/images/mvi-mosby3/mvp-business-logic.png)

当一些事件发生时（比如用户点击了`View1`按钮），`Presenter`将信息下沉到业务逻辑。因为其它的`Presenter`观察了相同的业务逻辑，因此它们从业务逻辑中接收到了同样变化的通知（`Model`被更新了）。

![](http://hannesdorfmann.com/images/mvi-mosby3/mvp-business-logic2.png)

关于这一点，我们已经在 [第一章节](https://github.com/qingmei2/android-programming-profile/blob/master/src/Android-MVI/%5B%E8%AF%91%5D%E4%BD%BF%E7%94%A8MVI%E6%89%93%E9%80%A0%E5%93%8D%E5%BA%94%E5%BC%8FAPP%5B%E4%B8%80%5D%3AModel%E5%B1%82%E5%88%B0%E5%BA%95%E4%BB%A3%E8%A1%A8%E4%BB%80%E4%B9%88.md) 讨论了 **单向数据流** 的原理的重要性。

让我们通过一个真实的案例实现它：在我们的购物`App`中，我们能够将商品加入购物车，此外，有这样一个页面，我们可以看到购物车商品的内容，并且能够一次选择或者删除多个商品条目：

我们如果能够将这样一个复杂的界面分割成多个小的页面的话就太棒了，这样UI组件
