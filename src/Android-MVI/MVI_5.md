# [译]使用MVI打造响应式APP(五):轻而易举地Debug

> 原文：[REACTIVE APPS WITH MODEL-VIEW-INTENT - PART5 - DEBUGGING WITH EASE](http://hannesdorfmann.com/android/mosby3-mvi-5)  
作者：[Hannes Dorfmann](http://hannesdorfmann.com)  
译者：[却把清梅嗅](https://github.com/qingmei2)  

前文我们探讨了`Model-View-Intent (MVI) `架构模式及其相关特性，在 [第一篇文章](https://github.com/qingmei2/android-programming-profile/blob/master/src/Android-MVI/%5B%E8%AF%91%5D%E4%BD%BF%E7%94%A8MVI%E6%89%93%E9%80%A0%E5%93%8D%E5%BA%94%E5%BC%8FAPP%5B%E4%B8%80%5D%3AModel%E5%B1%82%E5%88%B0%E5%BA%95%E4%BB%A3%E8%A1%A8%E4%BB%80%E4%B9%88.md) 中，我们谈到了 **单项数据流的重要性** 和 **应用状态应该被业务逻辑驱动**。本文我们将展示这种架构模式会怎样回报开发者，它可以让开发者在开发过程中更轻而易举进行debug。

遇到过这样的情况嘛？你得到了一个崩溃的报告，但是你无法复现这个`BUG`。听起来似曾相识？我也是！在花了很多时间查看堆栈跟踪和项目的源码后，最终我选择了放弃——关闭了这个`issue`，并提交了一个类似 **无法复现** 或者 **某个Android生产商的某种特定的机型导致的特殊错误** 的备注。

以我们的购物`App`举例来说，在`Home`界面，用户以某种方式进行下拉刷新，但不知道为什么，崩溃报告告诉我，当用户执行下拉刷新获取最新数据的操作时，应用抛出了一个`NullPointerException`。

因此，作为开发人员，您启动`App`并尝试在`Home`界面进行下拉刷新，但`App`并没有崩溃, 它按照预期正常地运行。然后您开始仔细检查自己的代码，但是就是找不到哪里会导致`NullPointerException`的发生。你打开了`debug`模式，一行一行逐步执行该界面相关的代码，但`App`仍然正常的运行—— 到底怎么样才能让它在下拉刷新时崩溃？

问题的根本在于你不能在`App`崩溃发生之前复现状态，如果遇到崩溃的用户可以在崩溃报告中提供他`App`的状态（在崩溃发生之前）以及堆栈跟踪，那不是很棒吗？

通过 **单向数据流** 和 **Model-View-Intent** ，这简直轻而易举。

在 **用户执行所有Intent** 和 **界面对Model进行渲染时**，我们很方便地能够将它们进行打印，让我们通过在`HomePresenter`中添加`Log`来为`Home`界面执行这样的操作（具体代码请参考 [第三节](https://github.com/qingmei2/android-programming-profile/blob/master/src/Android-MVI/%5B%E8%AF%91%5D%E4%BD%BF%E7%94%A8MVI%E6%89%93%E9%80%A0%E5%93%8D%E5%BA%94%E5%BC%8FAPP%5B%E4%B8%89%5D%3AStateReducer.md)，该小节我们针对状态折叠器进行了探讨）。

在以下代码片段中，我们使用`Crashlytics`（译者注：一种崩溃报告工具），使用其它的崩溃报告工具也是一样的：

```java
class HomePresenter extends MviBasePresenter<HomeView, HomeViewState> {

  private final HomeViewState initialState; // Show loading indicator

  public HomePresenter(HomeViewState initialState){
    this.initialState = initialState;
  }

  @Override protected void bindIntents() {

    Observable<PartialState> loadFirstPage = intent(HomeView::loadFirstPageIntent)
          .doOnNext(intent -> Crashlytics.log("Intent: load first page"))
          .flatmap(...); // 加载数据的业务逻辑

    Observable<PartialState> pullToRefresh = intent(HomeView::pullToRefreshIntent)
          .doOnNext(intent -> Crashlytics.log("Intent: pull-to-refresh"))
          .flatmap(...); // 加载数据的业务逻辑

    Observable<PartialState> nextPage = intent(HomeView::loadNextPageIntent)
          .doOnNext(intent -> Crashlytics.log("Intent: load next page"))
          .flatmap(...); // 加载数据的业务逻辑

    Observable<PartialState> allIntents = Observable.merge(loadFirstPage, pullToRefresh, nextPage);
    Observable<HomeViewState> stateObservable = allIntents
          .scan(initialState, this::viewStateReducer) // 对状态进行折叠
          .doOnNext(newViewState -> Crashlytics.log( "State: "+gson.toJson(newViewState) ));

    subscribeViewState(stateObservable, HomeView::render); // 展示新的状态
  }

  private HomeViewState viewStateReducer(HomeViewState previousState, PartialState changes){
    ...
  }
}
```

通过`RxJava`的 **.doOnNext()** 操作符，我们可以很轻松将每个`intent`和每个`intent`的`result`——也就是即将渲染在`view`层上的状态进行打印。

我们将`view`的状态序列化为json字符串，现在，我们的崩溃报告变成了这样：

![](http://hannesdorfmann.com/images/mvi-mosby3/crashlytics-mvi-logs.png)

现在来看看这些日志，我们不仅能看到崩溃发生之前的最后一个状态，而且还能看到用户达到这个状态所经历的完整历史记录——为了保证可读性，我将`data`字段内的内容替换为了[...]：

* 1.用户启动了`App`，通过加载首页数据的`intent`,这样`loadingFirstPage`的值为`true`，使得加载指示器展示了出来，同时数据也被加载完毕（data[…]）。

* 2.接下来用户滚动列表，并达到了列表的底部，这触发了加载下一页数据的`intent`，并开始加载更多的数据（分页），这也导致了`loadingNextPage`状态的改变，它的值变成了`true`。

* 3.一旦分页数据被加载成功，`loadingNextPage`状态改变成了`false`,用户再次重复操作达到了列表的底部，并又一次出发了触发了加载下一页数据的`intent`。

* 4.接下来用户开始尝试下拉刷新的`intent`,这导致`loadingPullToRefresh`状态变更为了`true`，然后，`App`突然发生了崩溃—— 这之后就没有更多日志了。

这些信息如何帮助我们解决这个bug呢？显然，我们知道用户触发了哪些操作，因此我们完全可以手动复现这个崩溃。此外，因为我们将`App`的状态用`json`进行表现，因此我们可以简单地使用最后一个状态，反序列化json并将此状态作为我们的初始状态来修复该错误：

```Java
String json ="  {\"data\":[...],\"loadingFirstPage\":false,\"loadingNextPage\":false,\"loadingPullToRefresh\":false} ";
HomeViewState stateBeforeCrash = gson.fromJson(json, HomeViewState.class);
HomePresenter homePresenter = new HomePresenter(stateBeforeCrash);
```

接下来我们打开了`Debug`调试工具，并尝试触发下拉刷新的`intent`,事实证明，如果用户向下滚动页面2次，则没有更多数据可用，并且我们的`App`并没有进行相应的处理，因此后续的下拉刷新操作导致了崩溃。

## 结语

一个应用状态随时随地 **可快照** 的`App`可以使我们开发人员的生活更加轻松。我们不仅能够轻松的 **复现崩溃**，而且可以将状态进行序列化来 **编写回归测试**，并且这几乎没有什么成本。

请记住，这些便利只有在`App`的状态遵循 **单项数据流** 、**不可变**、**纯函数** 的原则的情况下才能享受到（即被业务逻辑驱动），`Model-View-Intent`让我们偏向了这种思想流派，而这个架构模式中有一个非常棒并且有效的额外的效果，那就是本文所提到的构建了一个 **可快照** 的`App`。

**可快照** 的应用有什么缺陷呢？显然我们正在将`App`的状态序列化（比如通过`Gson`）.这增加了一些额外的计算资源的负荷，平均来算的话，状态第一次被`Gson`序列化大约需要30毫秒，因为`Gson`必须使用反射来扫描类，以确定必须序列化的字段。

在`Nexus 4`上，状态的连续序列化平均需要大约6毫秒。由于序列化在`.doOnNext()`中运行，虽然这通常在后台线程上运行，但的确是这样：我的`App`用户必须比其它应用的用户多等待6毫秒，才能在屏幕上看到新的状态。

我的观点是，这对于用户来说也许并不明显，但是对状态进行 **快照** 的一个问题是，在崩溃时，崩溃报告工具从用户设备上传到其服务器的数据量要大得多—— 如果用户通过wifi连接，这无关痛痒，但如果用户处于移动网络下则可能会有一定的争议。

最后，将状态附加在崩溃报告中时，您可能会泄漏用户的一些敏感的数据。针对这个问题，一个方案是不序列化敏感数据，但这可能导致连接到崩溃报告的状态不完整（因此这些报告可能几乎无用），另外一个方案则是将敏感数据进行加密——但这可能需要一些额外的CPU占用。

总结一下：我个人认为这样 **可快照** 的`App`有很多优点，但是，你可能需要做出一些权衡。也许您开始为内部版本或beta版本启用`App`快照，以衡量它其产生的作用。

---

## 系列目录

**《使用MVI打造响应式APP》原文**  

> * [Part 1: Model
](http://hannesdorfmann.com/android/mosby3-mvi-1)  
> * [Part 2: View and Intent](http://hannesdorfmann.com/android/mosby3-mvi-2)  
> * [Part 3: State Reducer](http://hannesdorfmann.com/android/mosby3-mvi-3)  
> * [Part 4: Independent UI Components
](http://hannesdorfmann.com/android/mosby3-mvi-4)  
> * [Part 5: Debugging with ease
](http://hannesdorfmann.com/android/mosby3-mvi-5)  
> * [Part 6: Restoring State
](http://hannesdorfmann.com/android/mosby3-mvi-6)  
> * [Part 7: Timing (SingleLiveEvent problem)
](http://hannesdorfmann.com/android/mosby3-mvi-7)  
> * [Part 8: In-App Navigation
](http://hannesdorfmann.com/android/mosby3-mvi-8)  

**《使用MVI打造响应式APP》译文**  
> * [[译]使用MVI打造响应式APP(一):Model到底是什么](https://github.com/qingmei2/android-programming-profile/blob/master/src/Android-MVI/%5B%E8%AF%91%5D%E4%BD%BF%E7%94%A8MVI%E6%89%93%E9%80%A0%E5%93%8D%E5%BA%94%E5%BC%8FAPP%5B%E4%B8%80%5D%3AModel%E5%B1%82%E5%88%B0%E5%BA%95%E4%BB%A3%E8%A1%A8%E4%BB%80%E4%B9%88.md)  
> * [[译]使用MVI打造响应式APP[二]:View层和Intent层](https://github.com/qingmei2/android-programming-profile/blob/master/src/Android-MVI/%5B%E8%AF%91%5D%E4%BD%BF%E7%94%A8MVI%E6%89%93%E9%80%A0%E5%93%8D%E5%BA%94%E5%BC%8FAPP%5B%E4%BA%8C%5D%3AView%E5%B1%82%E5%92%8CIntent%E5%B1%82.md)  
> * [[译]使用MVI打造响应式APP[三]:状态折叠器](https://github.com/qingmei2/android-programming-profile/blob/master/src/Android-MVI/%5B%E8%AF%91%5D%E4%BD%BF%E7%94%A8MVI%E6%89%93%E9%80%A0%E5%93%8D%E5%BA%94%E5%BC%8FAPP%5B%E4%B8%89%5D%3AStateReducer.md)  
> * [[译]使用MVI打造响应式APP[四]:独立性UI组件](https://github.com/qingmei2/android-programming-profile/blob/master/src/Android-MVI/%5B%E8%AF%91%5D%E4%BD%BF%E7%94%A8MVI%E6%89%93%E9%80%A0%E5%93%8D%E5%BA%94%E5%BC%8FAPP%5B%E5%9B%9B%5D%3AIndependentUIComponents.md)  
> * [[译]使用MVI打造响应式APP[五]:轻而易举地Debug](https://github.com/qingmei2/android-programming-profile/blob/master/src/Android-MVI/%5B%E8%AF%91%5D%E4%BD%BF%E7%94%A8MVI%E6%89%93%E9%80%A0%E5%93%8D%E5%BA%94%E5%BC%8FAPP%5B%E4%BA%94%5D%3ADebuggingWithEase.md)
> * [[译]使用MVI打造响应式APP[六]:恢复状态](https://github.com/qingmei2/android-programming-profile/blob/master/src/Android-MVI/%5B%E8%AF%91%5D%E4%BD%BF%E7%94%A8MVI%E6%89%93%E9%80%A0%E5%93%8D%E5%BA%94%E5%BC%8FAPP%5B%E5%85%AD%5D%3ARestoringState.md)
> * [[译]使用MVI打造响应式APP[七]:掌握时机(SingleLiveEvent问题)](https://github.com/qingmei2/android-programming-profile/blob/master/src/Android-MVI/%5B%E8%AF%91%5D%E4%BD%BF%E7%94%A8MVI%E6%89%93%E9%80%A0%E5%93%8D%E5%BA%94%E5%BC%8FAPP%5B%E4%B8%83%5D%3ATiming%2CSingleLiveEventProblem.md)
> * [[译]使用MVI打造响应式APP[八]:导航](https://github.com/qingmei2/android-programming-profile/blob/master/src/Android-MVI/%5B%E8%AF%91%5D%E4%BD%BF%E7%94%A8MVI%E6%89%93%E9%80%A0%E5%93%8D%E5%BA%94%E5%BC%8FAPP%5B%E5%85%AB%5D%3ANavigation.md)  

**《使用MVI打造响应式APP》实战**  
> * [实战：使用MVI打造响应式&函数式的Github客户端](https://github.com/qingmei2/MVI-Rhine)

---

## 关于我

Hello，我是[却把清梅嗅](https://github.com/qingmei2)，如果您觉得文章对您有价值，欢迎 ❤️，也欢迎关注我的[博客](https://www.jianshu.com/u/df76f81fe3ff)或者[Github](https://github.com/qingmei2)。

如果您觉得文章还差了那么点东西，也请通过**关注**督促我写出更好的文章——万一哪天我进步了呢？

* [我的Android学习体系](https://github.com/qingmei2/android-programming-profile)
* [关于文章纠错](https://github.com/qingmei2/Programming-life/blob/master/error_collection.md)
* [关于知识付费](https://github.com/qingmei2/Programming-life/blob/master/appreciation.md)
