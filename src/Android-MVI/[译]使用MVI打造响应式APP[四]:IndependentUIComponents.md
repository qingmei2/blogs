# [译]使用MVI打造响应式APP(四):独立性UI组件

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

![](https://user-gold-cdn.xitu.io/2019/3/15/1697fd47dca8940e?imageslim)

我们如果能够将这样一个复杂的界面分割成更多 **精巧、独立且可复用的UI组件** 的话就太棒了。以`Toolbar`为例，它展示了被选中条目的数量，以及`RecyclerView`展示了购物车里条目的列表。

```xml
<LinearLayout>
  <com.hannesdorfmann.SelectedCountToolbar
      android:id="@+id/selectedCountToolbar"
      android:layout_width="match_parent"
      android:layout_height="wrap_content"
      />

  <com.hannesdorfmann.ShoppingBasketRecyclerView
      android:id="@+id/shoppingBasketRecyclerView"
      android:layout_width="match_parent"
      android:layout_height="0dp"
      android:layout_weight="1"
      />
</LinearLayout>
```

但是这些组件之间如何保持相互的通讯呢？很明显每个组件都有它自己的`Presenter`:**`SelectedCountPresenter`** 和 **`ShoppingBasketPresenter`**。这属于父子关系吗？不，它们仅仅是观察了同一个`Model`，该`Model`根据在的逻辑代码中进行更新:

![](http://hannesdorfmann.com/images/mvi-mosby3/shoppingcart-businesslogic.png)

```Java
public class SelectedCountPresenter
    extends MviBasePresenter<SelectedCountView, Integer> {

  private ShoppingCart shoppingCart;

  public SelectedCountPresenter(ShoppingCart shoppingCart) {
    this.shoppingCart = shoppingCart;
  }

  @Override protected void bindIntents() {
    subscribeViewState(shoppingCart.getSelectedItemsObservable(), SelectedCountView::render);
  }
}


class SelectedCountToolbar extends Toolbar implements SelectedCountView {

  ...

  @Override public void render(int selectedCount) {
   if (selectedCount == 0) {
     setVisibility(View.VISIBLE);
   } else {
       setVisibility(View.INVISIBLE);
   }
 }
}
```

**ShoppingBasketRecyclerView** 的代码和上述代码的实现非常类似，因此本文不对其进行展示。然而，如果我们认真去观察这段代码，你会发现`SelectedCountPresenter`和`ShoppingCart`有一定的耦合。

我们完全有可能会在其它的页面去复用这个UI组件，因此我们需要移除这个依赖的关系以达到复用该组件的目的。重构其实很简单：`presenter`持有一个 **`Observable<Integer>`** 作为`Model`代替之前构造器中所需要的`ShoppingCart`：

```java
public class SelectedCountPresenter
    extends MviBasePresenter<SelectedCountView, Integer> {

  private Observable<Integer> selectedCountObservable;

  public SelectedCountPresenter(Observable<Integer> selectedCountObservable) {
    this.selectedCountObservable = selectedCountObservable;
  }

  @Override protected void bindIntents() {
    subscribeViewState(selectedCountObservable, SelectedCountToolbarView::render);
  }
}
```

There you go (原文为法语，大概意思是“就是这样”)，每当我们需要显示当前选择的条目数量时，我们就可以使用`SelectedCountToolbar`组件——这可以代表`ShoppingCart`中的条目数，也可以表示在`App`中的完全不同的上下文环境和页面中。

此外，此UI组件可以放入独立的库中，并在另一个`App`（如相册应用程序）中使用，以显示所选照片的​​数量：

```Java
Observable<Integer> selectedCount = photoManager.getPhotos()
    .map(photos -> {
       int selected = 0;
       for (Photo item : photos) {
         if (item.isSelected()) selected++;
       }
       return selected;
    });

return new SelectedCountToolbarPresnter(selectedCount);
```

## 结语

本文的目的是证明通常情况下，代码的设计中根本不需要 **父子关系** ，它们仅需要通过简单的对相同业务逻辑进行观察就能实现。

不需要`EventBus`，不需要从上层的`Activity`或者`Fragment`中调用`findViewById()`，不需要`presenter.getParentPresenter()`或者其它的解决方案。仅使用 **观察者模式** 就够了。借助于`RxJava`——它本身也是基于观察者模式思想的体现，我们就能够轻而易举构建这样响应式的UI组件。

### 额外的思考

与`MVP`或`MVVM`相比，`MVI`的实现过程中，我们被迫（通过积极的方式）使用业务逻辑驱动某个组件的状态。因此，具有更多`MVI`经验的开发人员可以得出以下结论：

> 如果`View`的状态是另一个组件的`Model`怎么办？如果一个组件的`ViewState`的变更是另一个组件的`Intent`怎么办?

举个例子：

```java
Observable<Integer> selectedItemCountObservable =
        shoppingBasketPresenter
           .getViewStateObservable()
           .map(items -> {
              int selected = 0;
              for (ShoppingCartItem item : items) {
                if (item.isSelected()) selected++;
              }
              return selected;
            });

Observable<Boolean> doSomethingBecauseOtherComponentReadyIntent =
        shoppingBasketPresenter
          .getViewStateObservable()
          .filter(state -> state.isShowingData())
          .map(state -> true);

return new SelectedCountToolbarPresenter(
              selectedItemCountObservable,
              doSomethingBecauseOtherComponentReadyIntent);
```

乍一看，这似乎是一种可行的方案，但它不是父子关系的变体吗？当然不是，这并非传统分层的父子关系，也许将其比喻为洋葱更为恰当（洋葱的内层为外层提供了一种状态）。

但是，这依然是一种耦合的关系，不是吗？我还没有下定决心，但现在我认为避免这种洋葱般的关系更好。如果您有不同意见，请在下面留言，我很期待您的观点。

**--------------------------广告分割线------------------------------**

**《使用MVI打造响应式APP》翻译系列**  
> * [[译]使用MVI打造响应式APP(一):Model到底是什么](https://github.com/qingmei2/android-programming-profile/blob/master/src/Android-MVI/%5B%E8%AF%91%5D%E4%BD%BF%E7%94%A8MVI%E6%89%93%E9%80%A0%E5%93%8D%E5%BA%94%E5%BC%8FAPP%5B%E4%B8%80%5D%3AModel%E5%B1%82%E5%88%B0%E5%BA%95%E4%BB%A3%E8%A1%A8%E4%BB%80%E4%B9%88.md)  
> * [[译]使用MVI打造响应式APP[二]:View层和Intent层](https://github.com/qingmei2/android-programming-profile/blob/master/src/Android-MVI/%5B%E8%AF%91%5D%E4%BD%BF%E7%94%A8MVI%E6%89%93%E9%80%A0%E5%93%8D%E5%BA%94%E5%BC%8FAPP%5B%E4%BA%8C%5D%3AView%E5%B1%82%E5%92%8CIntent%E5%B1%82.md)  
> * [[译]使用MVI打造响应式APP[三]:状态折叠器](https://github.com/qingmei2/android-programming-profile/blob/master/src/Android-MVI/%5B%E8%AF%91%5D%E4%BD%BF%E7%94%A8MVI%E6%89%93%E9%80%A0%E5%93%8D%E5%BA%94%E5%BC%8FAPP%5B%E4%B8%89%5D%3AStateReducer.md)  
> * [[译]使用MVI打造响应式APP[四]:独立性UI组件](https://github.com/qingmei2/android-programming-profile/blob/master/src/Android-MVI/%5B%E8%AF%91%5D%E4%BD%BF%E7%94%A8MVI%E6%89%93%E9%80%A0%E5%93%8D%E5%BA%94%E5%BC%8FAPP%5B%E5%9B%9B%5D%3AIndependentUIComponents.md)  
> * [[译]使用MVI打造响应式APP[五]:DebuggingWithEase](https://github.com/qingmei2/android-programming-profile/blob/master/src/Android-MVI/%5B%E8%AF%91%5D%E4%BD%BF%E7%94%A8MVI%E6%89%93%E9%80%A0%E5%93%8D%E5%BA%94%E5%BC%8FAPP%5B%E4%BA%94%5D%3ADebuggingWithEase.md)
> * [[译]使用MVI打造响应式APP[六]:RestoringState](https://github.com/qingmei2/android-programming-profile/blob/master/src/Android-MVI/%5B%E8%AF%91%5D%E4%BD%BF%E7%94%A8MVI%E6%89%93%E9%80%A0%E5%93%8D%E5%BA%94%E5%BC%8FAPP%5B%E5%85%AD%5D%3ARestoringState.md)
> * [[译]使用MVI打造响应式APP[七]:Timing,SingleLiveEventProblem](https://github.com/qingmei2/android-programming-profile/blob/master/src/Android-MVI/%5B%E8%AF%91%5D%E4%BD%BF%E7%94%A8MVI%E6%89%93%E9%80%A0%E5%93%8D%E5%BA%94%E5%BC%8FAPP%5B%E4%B8%83%5D%3ATiming%2CSingleLiveEventProblem.md)
> * [[译]使用MVI打造响应式APP[八]:Navigation](https://github.com/qingmei2/android-programming-profile/blob/master/src/Android-MVI/%5B%E8%AF%91%5D%E4%BD%BF%E7%94%A8MVI%E6%89%93%E9%80%A0%E5%93%8D%E5%BA%94%E5%BC%8FAPP%5B%E5%85%AB%5D%3ANavigation.md)  

**《使用MVI打造响应式APP》实战系列**  
> * [实战：使用MVI打造响应式&函数式的Github客户端](https://github.com/qingmei2/MVI-Rhine)

## 关于我

Hello，我是[却把清梅嗅](https://github.com/qingmei2)，如果您觉得文章对您有价值，欢迎 ❤️，也欢迎关注我的[博客](https://www.jianshu.com/u/df76f81fe3ff)或者[Github](https://github.com/qingmei2)。

如果您觉得文章还差了那么点东西，也请通过**关注**督促我写出更好的文章——万一哪天我进步了呢？

* [我的Android学习体系](https://github.com/qingmei2/android-programming-profile)
* [关于文章纠错](https://github.com/qingmei2/Programming-life/blob/master/error_collection.md)
* [关于知识付费](https://github.com/qingmei2/Programming-life/blob/master/appreciation.md)
