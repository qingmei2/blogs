# [译]使用MVI打造响应式APP(六):恢复状态

> 原文：[REACTIVE APPS WITH MODEL-VIEW-INTENT - PART6 - RESTORING STATE](http://hannesdorfmann.com/android/mosby3-mvi-6)  
作者：[Hannes Dorfmann](http://hannesdorfmann.com)  
译者：[却把清梅嗅](https://github.com/qingmei2)  

在前几篇文章中，我们讨论了`Model-View-Intent(MVI)`和单向数据流的重要性，这极大简化了状态的恢复，那么其过程和原理是什么呢，本文我们针对这个问题进行探讨。

我们将针对2个场景进行探讨：

* 在内存中恢复状态（比如当屏幕方向发生改变）
* 持久化恢复状态（比如从`Bundle`中获取之前在`Activity.onSaveInstanceState()`保存的状态）

##  内存中

这种情况处理起来非常简单。我们只需要保持我们的`RxJava`流随着时间的推移从`Android`生命周期组件（即`Activity`，`Fragment`甚至`ViewGroups`）种发射新的状态。

比如`Mosby`的 **`MviBasePresenter`** 类在内部就使用了类似这样的`RxJava`的流：使用 **PublishSubject** 发射`intent`，以及使用 **BehaviorSubject** 对`View`进行渲染。对此，在 [第二部分](https://github.com/qingmei2/android-programming-profile/blob/master/src/Android-MVI/%5B%E8%AF%91%5D%E4%BD%BF%E7%94%A8MVI%E6%89%93%E9%80%A0%E5%93%8D%E5%BA%94%E5%BC%8FAPP%5B%E4%BA%8C%5D%3AView%E5%B1%82%E5%92%8CIntent%E5%B1%82.md) 中我已经阐述了是如何实现的。其主要思想是`MviBasePresenter`是一个和`View`生命周期隔离的组件，因此它能够被`View`脱离和附着。在`Mosby`中，当`View`被永久销毁时，`Presenter`被`destroyed`（垃圾收集）。同样，这只是`Mosby`的一个实现细节，您的MVI实现可能完全不同。

重要的是，像`Presenter`这样的组件存活在`View`的生命周期之外，因为这样很容易处理`View`脱离和附着的事件。每当`View`（重新）依附到`Presenter`时，我们只需调用`view.render(previousState)`（因此Mosby内部使用了`BehaviorSubject`）。

这只是处理屏幕方向改变的一种处理方案，它同样适用于返回栈导航中。例如，`Fragment`在返回栈中，我们如果从返回栈中返回，我们可以简单的再次调用`view.render(previousState)`，并且，`view`也会显示正确的状态。

事实上，即使没有`View`对其进行依附，状态也依然会被更新，因为`Presenter`存活在`View`的生命周期之外，并被保存在`RxJava`流中。设想如果没有`View`附着，则会收到一个更改数据（部分状态）的推送通知，同样，每当`View`重新附着时，最新状态（包含来自推送通知的更新数据）将被移交给`View`进行渲染。

## 持久化状态

这种场景在`MVI`这种单向数据流模式下也很简单。现在我们希望`View`层的状态不仅仅存在于内存中，即使进程终止也能够对其持有。`Android`中通常的一种解决方案是通过调用`Activity.onSaveInstanceState(Bundle)`去保存状态。

与`MVP`、`MVVM`不同的是，在`MVI`中你持有了代表状态的`Model`,`View`有一个`render(state)`方法来记录最新的状态，这让持有最后一个状态变得简单。因此，显然易见的是打包和存储状态到一个`bundle`下面，并且之后恢复它：

```Java
class MyActivity extends Activity implements MyView {

  private final static KEY_STATE = "MyStateKey";
  private MyViewState lastState;

  @Override
  public void render(MyState state) {
    lastState = state;
    ... // 更新UI控件
  }

  @Override
  public void onSaveInstanceState(Bundle out){
    out.putParcelable(KEY_STATE, lastState);
  }

  @Override
  public void onCreate(Bundle saved){
    super.onCreate(saved);
    MyViewState initialState = null;
    if (saved != null){
      initialState = saved.getParcelable(KEY_STATE);
    }

    presenter = new MyPresenter( new MyStateReducer(initialState) ); // With dagger: new MyDaggerModule(initialState)
  }
  ...
}
```

我想你已得要领，请注意，在`onCreate()`中我们并不直接调用`view.render(initialState)`, 我们让初始状态的逻辑下沉到状态管理的地方： **状态折叠器**（请参考[第三部分](https://github.com/qingmei2/android-programming-profile/blob/master/src/Android-MVI/%5B%E8%AF%91%5D%E4%BD%BF%E7%94%A8MVI%E6%89%93%E9%80%A0%E5%93%8D%E5%BA%94%E5%BC%8FAPP%5B%E4%B8%89%5D%3AStateReducer.md)）,我们将它与`.scan（initialState，reducerFunction）`搭配使用。

## 结语

与其他模式相比，使用单向数据流和表示状态的`Model`，许多与状态相关的东西更容易实现。但是，我通常不会在我的`App`中将状态持久化，两个原因：首先，`Bundle`有大小限制，因此你不能将任意大的状态放入`bundle`中（或者你可以将状态保存到文件或像`Realm`这样的对象存储中）；其次，我们只讨论了如何序列化和反序列化状态，但这不一定与恢复状态相同。

例如：假设我们有一个`LCE`（加载内容错误）视图，它会在加载数据时显示一个指示器，并在完成加载后显示条目列表，因此状态就类似`MyViewState.LOADING`。让我们假设加载需要一些时间，而就在此时进程刚好被终止了（比如突然一个电话打了进来，导致电话应用占据了前台）。

如果我们仅仅将`MyViewState.LOADING`进行序列化并在之后进行反序列化操作对状态进行恢复，我们的**状态折叠器**会调用`view.render(MyViewState.LOADING)`，目前为止这是正确的，但实际上我们 **永远不会通过这个状态对网络进行请求加载数据**。

如您所见，序列化和反序列化状态与状态恢复不同，这可能需要一些额外的步骤来增加复杂性（当然对于`MVI`来说这实现起来同样比其它模式更简单），当重新创建`View`时，包含某些数据的反序列化状态可能会过时，因此您可能必须刷新（加载数据）。

在我研究过的大多数应用程序中，我发现相比之下这种方案更简单且友好：即，将状态仅仅保存在内存中,并且在进程死亡后以空的初始状态启动，好像应用程序将首次启动一样。理想情况下，`App`具有对缓存和离线的支持，因此在进程终止后加载数据的速度也会很快。

这最终导致了我和其它`Android`开发者针对一个问题进行了激烈的辩论：

> 如果我使用缓存或存储，我已经拥有了一个存活于`Android`组件生命周期之外的组件，而且我不再需要去处理相关的状态存储问题，并且`MVI`毫无意义，对吗？

这其中大多数`Android`开发者推荐 `Mike Nakhimovich` 发表的 [《Presenter 不是为了持久化》](https://hackernoon.com/presenters-are-not-for-persisting-f537a2cc7962)这篇文章介绍的 [NyTimes Store](https://github.com/NYTimes/Store),这是一个数据加载和缓存库。遗憾的是，那些开发人员不明白 **加载数据和缓存不是状态管理**。例如，如果我必须从缓存或存储中加载数据呢？

最后，类似[NyTimes Store](https://github.com/NYTimes/Store)的库帮助我们处理进程终止了吗？显然没有，因为进程随时都有可能被终止。我们能做的仅仅是祈祷`Android`操作系统不要杀死我们的进程，因为我们还有一些需要通过`Service`做的事（这也是一个能够不生存于其它android组件生命周期的组件），或者我们可以通过使用`RxJava`而不再需要`Android Service`了，这可行吗？

我们将在下一章节探讨关于`android services`、`RxJava`以及`MVI`，敬请期待。

##### 剧透：我认为我们确实需要服务。

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
