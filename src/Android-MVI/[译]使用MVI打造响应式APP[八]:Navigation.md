# [译]使用MVI打造响应式APP(八):导航

> 原文：[REACTIVE APPS WITH MODEL-VIEW-INTENT - PART 8 - NAVIGATION](http://hannesdorfmann.com/android/mosby3-mvi-8)  
作者：[Hannes Dorfmann](http://hannesdorfmann.com)  
译者：[却把清梅嗅](https://github.com/qingmei2)  


在[上一篇博客](http://hannesdorfmann.com/android/coordinators-android)中，我们探讨了协调模式是如何在`Android`中应用的。这次我想展示如何在`Model-View-Intent`中使用它。

如果您还不知道协调器模式是什么，我强烈建议您回过头来阅读[上文内容](http://hannesdorfmann.com/android/coordinators-android)。

在`MVI`中应用此模式与`MVVM`或`MVP`没有太大区别：我们将lambda作为导航的回调传递给我们的`MviBasePresenter`。有趣的是我们如何在状态驱动的架构中触发这些回调？我们来看一个具体的例子：

```Kotlin
class FooPresenter(
  private var navigationCallback: ( () -> Unit )?
) : MviBasePresenter<FooView> {  
  lateinit var disposable : Disposable
  override fun bindIntents(){
    val intent1 = ...
    val intent2 = ...
    val intents = Observable.merge(intent1, intent2)

    val state = intents.switchMap { ... }

    // 这里就是有趣的部分
    val sharedState = state.share()
    disposable = sharedState.filter{ state ->
      state is State.Foo
    }.subscribe { navigationCallback!!() }

    subscribeViewState(sharedState, FooView::render)
  }

  override fun unbindIntents(){
    disposable.dispose() // disposable 导航
    navigationCallback = null // 避免内存泄漏
  }
}
```

其思想是：通过`RxJava`的 **share()** 操作符，我们对通常用来对`View`层渲染状态的`Observable`进行复用，再加上通过与 **.filter()** 操作符的组合使用，达到能够监听到确定的状态，这之后，当我们观察到该状态时，触发对应的导航操作，然后协调器模式就像我之前的博客文章中描述的那样进行工作。

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
