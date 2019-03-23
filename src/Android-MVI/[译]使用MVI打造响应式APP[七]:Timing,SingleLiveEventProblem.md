> 原文：[REACTIVE APPS WITH MODEL-VIEW-INTENT - PART7 - TIMING (SINGLELIVEEVENT PROBLEM)](http://hannesdorfmann.com/android/mosby3-mvi-7)  
作者：[Hannes Dorfmann](http://hannesdorfmann.com)  
译者：[却把清梅嗅](https://github.com/qingmei2)  

在之前的文章中，我们探讨了正确状态管理的重要性，以及我为什么认为使用类似 [Github上Google架构组件的这个repo](https://github.com/googlesamples/android-architecture-components/issues/63) 中的 **SingleLiveEvent** 并不是一个好主意——这种解决方案只是隐藏了真正的潜在问题，那就是**状态管理**。本文我将会阐述`SingleLiveEvent`声称能解决的问题，在`Model-View-Intent`中如何通过状态管理正确地解决。

> 译者注：关于`SingleLiveEvent`的这个[issue](https://github.com/googlesamples/android-architecture-components/issues/63) 从17年讨论到19年至今还未close，各方大佬（还有google的巨佬）针对`SingleLiveEvent`进行了激烈的讨论，堪称Android论坛的一场神仙大战，非常值得一看。

当`error`发生时，**Snackbar** 将被展示—— 这是一个常见的场景可以用来描述这个问题。`Snackbar`并非一直展示，当错误信息被展示几秒钟之后它将消失，问题在于，我们如何模拟这种错误状态并控制`Snackbar`的消失呢？

通过下面的视频，你就能了解我在说什么：


![](https://user-gold-cdn.xitu.io/2019/3/23/169a9ba3c413752f?w=274&h=498&f=gif&s=389582)

这个示例显示了如何从 **CountriesRepository** 中加载国家的列表，当我们点击某个国家的条目时，程序将会跳转到第二个界面去展示“详情”（仅仅是国家的名称）。当我们返回到国家列表的界面时，我们希望和之前点击国家条目时的状态一致。

目前为止一切正常，但是当我们执行下拉刷新操作时，一个异常出现了，并通过在屏幕中展示一个`Snackbar`来展示错误信息。正如您在视频中看到的，当我们再次从国家详情返回国家列表时，`Snackbar`和相关的错误信息再次展示了出来，这和用户预期的并不一致，不是吗？

问题的根源是展示了错误的状态。Google基于`ViewModel`和`LiveData`的架构组件示例中使用了 **SingleLiveEvent** 来解决这个问题。其解决思路是：当`View`重新订阅了其`ViewModel`（当从“国家详情”界面返回时），`SingleLiveEvent`确保错误的状态不会再次被发射，这确实预防了`SnakeBar`的再次显示，但是这真的解决问题了吗？

## 时机就是一切（也适用于`SnakeBar`）

重申，我依然认为这种解决方案并不恰当，我们能做的更好吗？我认为合理的运用 **状态管理** 和 **单向的数据流** 是更好的答案，而`Model-View-Intent`架构模式遵循了这些规则。因此在`MVI`中我们如何解决`SnakeBar`的问题呢？首先，我们对`Model`状态进行定义：

```Java
public class CountriesViewState {

  // true意味着Progress将被展示
  boolean loading;

  // 被加载的国家列表
  List<String> countries;

  // true意味着`SwipeRefreshLayout`将被展示
  boolean pullToRefresh;

  // true意味着下拉刷新出现了error，SnakeBar将被展示
  boolean pullToRefreshError;
}
```

`MVI`的思想是`View`层同时只会展示一个不可变的`CountriesViewState`,因此当`pullToRefreshError`为`true`时`SnakeBar`将被展示，反之则会消失。

```Java
public class CountriesActivity extends MviActivity<CountriesView, CountriesPresenter>
    implements CountriesView {

  private Snackbar snackbar;
  private ArrayAdapter<String> adapter;

  @BindView(R.id.refreshLayout) SwipeRefreshLayout refreshLayout;
  @BindView(R.id.listView) ListView listView;
  @BindView(R.id.progressBar) ProgressBar progressBar;

   ...

  @Override public void render(CountriesViewState viewState) {
    if (viewState.isLoading()) {
      progressBar.setVisibility(View.VISIBLE);
      refreshLayout.setVisibility(View.GONE);
    } else {
      // 展示国家列表
      progressBar.setVisibility(View.GONE);
      refreshLayout.setVisibility(View.VISIBLE);
      adapter.setCountries(viewState.getCountries());
      refreshLayout.setRefreshing(viewState.isPullToRefresh());

      if (viewState.isPullToRefreshError()) {
        showSnackbar();
      } else {
        dismissSnackbar();
      }
    }
  }

  private void dismissSnackbar() {
    if (snackbar != null)
      snackbar.dismiss();
  }

  private void showSnackbar() {
    snackbar = Snackbar.make(refreshLayout, "An Error has occurred", Snackbar.LENGTH_INDEFINITE);
    snackbar.show();
  }
}
```

关键点在于 **Snackbar.LENGTH_INDEFINITE**，这意味着`Snackbar`将会一直显示直到我们主动控制其消失——我们不需要让`Android`系统控制它显示与否。

我们不会让`Android`系统将状态搞乱，也不会让系统为UI引入一个与业务逻辑状态不一致的状态。我们宁愿让业务逻辑将`CountriesViewState.pullToRefreshError`设置为`true`两秒钟，然后将其设置为`false`，而不愿使用`Snackbar.LENGTH_SHORT`来显示`Snackbar`两秒钟。

在`RxJava`中我们怎么做呢？我们可以使用 **Observable.timer()** 和 **startWith()** 操作符：

```Java
public class CountriesPresenter extends MviBasePresenter<CountriesView, CountriesViewState> {

  private final CountriesRepositroy repositroy = new CountriesRepositroy();

  @Override protected void bindIntents() {

    Observable<RepositoryState> loadingData =
        intent(CountriesView::loadCountriesIntent).switchMap(ignored -> repositroy.loadCountries());

    Observable<RepositoryState> pullToRefreshData =
        intent(CountriesView::pullToRefreshIntent).switchMap(
            ignored -> repositroy.reload().switchMap(repoState -> {
              if (repoState instanceof PullToRefreshError) {
                // 展示snakebar2秒中，然后dismiss它
                return Observable.timer(2, TimeUnit.SECONDS)
                    .map(ignoredTime -> new ShowCountries()) // 仅仅展示列表
                    .startWith(repoState); // repoState == PullToRefreshError
              } else {
                return Observable.just(repoState);
              }
            }));

    // 作为初始状态，展示加载中
    CountriesViewState initialState = CountriesViewState.showLoadingState();

    Observable<CountriesViewState> viewState = Observable.merge(loadingData, pullToRefreshData)
        .scan(initialState, (oldState, repoState) -> repoState.reduce(oldState))

    subscribeViewState(viewState, CountriesView::render);
  }
```

**CountriesRepositroy** 的 `reload()` 方法返回了一个 **Observable<RepoState\>**。`RepoState`(前文中叫做`PattialViewState`) 仅仅是个`POJO`类，用来表示`repository`是否取到数据，是成功的取到数据，或者产生了错误（[点击查看源码](https://github.com/sockeqwe/mvi-timing/blob/41095bdecf32c149c1d81b3d773937e7c08d4bdf/app/src/main/java/com/hannesdorfmann/mvisnackbar/RepositoryState.java)）。

这之后，我们使用[状态折叠器](http://hannesdorfmann.com/android/mosby3-mvi-3)去计算`View`的状态（通过`scan()`操作符），如果您已阅读我之前的MVI系列文章，那么这应该此曾相识， “新”的东西是：

```Java
repositroy.reload().switchMap(repoState -> {
  if (repoState instanceof PullToRefreshError) {
    // 展示snakebar2秒中，然后dismiss它
    return Observable.timer(2, TimeUnit.SECONDS)
        .map(ignoredTime -> new ShowCountries()) // 仅仅展示列表
        .startWith(repoState); // repoState == PullToRefreshError
  } else {
    return Observable.just(repoState);
  }
```

这段代码执行以下操作：如果我们得到了一个`error`(`repoState instanceof PullToRefreshError`)，我们会发射一个错误的状态（PullToRefreshError），这使得状态折叠器设置 **CountriesViewState.pullToRefreshError = true**。2秒后，`Observable.timer()`将发射`ShowCountries`状态，状态折叠器会设置 **CountriesViewState.pullToRefreshError = false**。

OK, 现在你可以看到`MVI`中我们如何显示和隐藏`SnakeBar`：

![](https://user-gold-cdn.xitu.io/2019/3/23/169a9bd5318c4252?w=274&h=498&f=gif&s=373839)

请记住这并非像是`SingleLiveEvent`这样的解决方案。这是正确的状态管理，View只是显示或“渲染”给定的状态。因此用户如果再次从“国家详情”中返回，`Snackbar`不再会被展示，因为状态在 **CountriesViewState.pullToRefreshError = false** 时已经发生了改变。

## 用户消除Snakebar

如果我们希望用户能够通过滑动操作主动消除`Snakebar`呢。这非常简单，**消除Snakebar** 本身也是改变状态的一种`intent`，要将它添加到目前的代码中，我们只需要确保定时器或者滑动的意图能够设置`CountriesViewState.pullToRefreshError = false`。

我们唯一需要处理的是，如果在计时结束之前出发了滑动解除的`intent`，我们必须结束定时器的计时行为，这听起来很复杂，但得益于`RxJava`的优秀`api`和操作符，这轻而易举：

```Java
Observable<Long> dismissPullToRefreshErrorIntent = intent(CountriesView::dismissPullToRefreshErrorIntent)

...

repositroy.reload().switchMap(repoState -> {
  if (repoState instanceof PullToRefreshError) {
    // 展示Snakebar，并在2秒后dismiss它
    return Observable.timer(2, TimeUnit.SECONDS)
        .mergeWith(dismissPullToRefreshErrorIntent) // 合并计时器和滑动dismiss的intent
        .take(1) // 二者只会触发其中一个
        .map(ignoredTime -> new ShowCountries()) // 展示列表
        .startWith(repoState); // repoState == PullToRefreshError
  } else {
    return Observable.just(repoState);
  }
```

![](https://user-gold-cdn.xitu.io/2019/3/23/169a9bdb1dea7875?w=406&h=720&f=gif&s=90051)

使用`mergeWith()`，我们将计时器和滑动消失的`intent`组合成一个`observable`，然后使用`take(1)`仅将它们中的第一个事件进行发射。如果在计时器计时结束之前滑动`Snakebar`，则取消计时器，反之则取消滑动消失的`intent`。

## 结语
现在让我们来尝试将UI搞乱，我们尝试下拉刷新、并在计时过程中手动取消`Snakebar`：

![](https://user-gold-cdn.xitu.io/2019/3/23/169a9be0f68504a0?w=406&h=720&f=gif&s=90051)

如你所见，无论我们如何尝试，都没有问题发生，由于 **单向数据流** 和 **业务逻辑驱动的状态**，`View`可以正确显示UI小部件（`View`层是无状态的，它从底层获取状态并只能对它进行展示）。比如，我们从未看到下拉刷新指示器和`Snakebar`同时显示（除了`Snackbar`退出过程中，两者的叠加情况）。

当然，`Snackbar`这个示例非常简单，但我认为它证明了像`Model-View-Intent`这样 **严格规范下对状态进行管理** 的架构模式的强大。不难想象这种模式对于更复杂的屏幕和使用场景同样也会非常棒。

本文的示例源码你可以从[这里](https://github.com/sockeqwe/mvi-timing)获取。


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
> * [[译]使用MVI打造响应式APP[八]:Navigation](https://github.com/qingmei2/android-programming-profile/blob/master/src/Android-MVI/%5B%E8%AF%91%5D%E4%BD%BF%E7%94%A8MVI%E6%89%93%E9%80%A0%E5%93%8D%E5%BA%94%E5%BC%8FAPP%5B%E5%85%AB%5D%3ANavigation.md)  

**《使用MVI打造响应式APP》实战**  
> * [实战：使用MVI打造响应式&函数式的Github客户端](https://github.com/qingmei2/MVI-Rhine)

---

## 关于我

Hello，我是[却把清梅嗅](https://github.com/qingmei2)，如果您觉得文章对您有价值，欢迎 ❤️，也欢迎关注我的[博客](https://www.jianshu.com/u/df76f81fe3ff)或者[Github](https://github.com/qingmei2)。

如果您觉得文章还差了那么点东西，也请通过**关注**督促我写出更好的文章——万一哪天我进步了呢？

* [我的Android学习体系](https://github.com/qingmei2/android-programming-profile)
* [关于文章纠错](https://github.com/qingmei2/Programming-life/blob/master/error_collection.md)
* [关于知识付费](https://github.com/qingmei2/Programming-life/blob/master/appreciation.md)
