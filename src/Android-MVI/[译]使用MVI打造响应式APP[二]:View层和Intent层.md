# [译]使用MVI打造响应式APP(二):View层和Intent层

> 原文：[REACTIVE APPS WITH MODEL-VIEW-INTENT - PART2 - VIEW AND INTENT](http://hannesdorfmann.com/android/mosby3-mvi-2)  
作者：[Hannes Dorfmann](http://hannesdorfmann.com)  
译者：[却把清梅嗅](https://github.com/qingmei2)  

在 [上文](https://github.com/qingmei2/android-programming-profile/blob/master/src/Android-MVI/%5B%E8%AF%91%5D%E4%BD%BF%E7%94%A8MVI%E6%89%93%E9%80%A0%E5%93%8D%E5%BA%94%E5%BC%8FAPP%5B%E4%B8%80%5D%3AModel%E5%B1%82%E5%88%B0%E5%BA%95%E4%BB%A3%E8%A1%A8%E4%BB%80%E4%B9%88.md) 中，我们探讨了对`Model`的定义、与 **状态** 的关系以及如何在通过良好地定义`Model`来解决一些`Android`开发中常见的问题。本文将通过 `Model-View-Intent` ，即`MVI`模式，继续我们的 **响应式App** 构建之旅。

如果您尚未阅读上一小节，则应在继续阅读本文之前阅读该部分。总结一下：以“传统的”`MVP`为例，请避免写出这样的代码：

```Java
class PersonsPresenter extends Presenter<PersonsView> {

  public void load(){
    getView().showLoading(true); // 展示一个 ProgressBar

    backend.loadPersons(new Callback(){
      public void onSuccess(List<Person> persons){
        getView().showPersons(persons); // 展示用户列表
      }

      public void onError(Throwable error){
        getView().showError(error); // 展示错误信息
      }
    });
  }
}
```

我们应该创建能够反映 **状态** 的`Model`，像这样：

```Java
class PersonsModel {
  // 在真实的项目中，需要定义为私有的
  // 并且我们需要通过getter和setter来访问它们
  final boolean loading;
  final List<Person> persons;
  final Throwable error;

  public(boolean loading, List<Person> persons, Throwable error){
    this.loading = loading;
    this.persons = persons;
    this.error = error;
  }
}
```

因此，`Presenter`层也应该像这样进行定义：

```Java
class PersonsPresenter extends Presenter<PersonsView> {

  public void load(){
    getView().render( new PersonsModel(true, null, null) ); // 展示一个 ProgressBar

    backend.loadPersons(new Callback(){
      public void onSuccess(List<Person> persons){
        getView().render( new PersonsModel(false, persons, null) );  // 展示用户列表
      }

      public void onError(Throwable error){
          getView().render( new PersonsModel(false, null, error) ); // 展示错误信息
      }
    });
  }
}
```

现在，仅需简单调用`View`层的`render(personsModel)`方法，`Model`就会被成功的渲染在屏幕上。在第一小节中我们同样探讨了 **单项数据流** 的重要性，同时您的业务逻辑应该驱动该`Model`。在正式将所有内容环环相扣连接之前，我们先简单了解一下`MVI`的核心思想。

## Model-View-Intent (MVI)

该模式最初被 [andrestaltz](https://twitter.com/andrestaltz) 在他写的`JavaScript`框架 [cycle.js](https://cycle.js.org/) 中所提出; 从理论(还有数学)上讲，我们这样对`Model-View-Intent`的定义进行描述：

![](http://hannesdorfmann.com/images/mvi/mvi-func2.png)

### 1.intent()

此函数接受来自用户的输入（即UI事件，比如点击事件）并将其转换为可传递给`Model()`函数的参数，该参数可能是一个简单的`String`对`Model`进行赋值，也可能像是`Object`这样复杂的数据结构。`intent`作为意图，标志着 **我们试图对`Model`进行改变**。

### 2.model()

`model()`函数将`intent()`函数的输出作为输入来操作`Model`，其函数输出是一个新的`Model`（状态发生了改变）。

**不要对已存在的`Model`对象进行修改，我们需要的是不可变**！对此，在上文中我们已经展示了一个计数器的具体案例，再次重申，不要修改已存在的`Model`！

根据`intent`所描述的变化，我们创建一个新的`Model`,请注意，`Model()`函数是唯一允许对`Model`进行创建的途径。然后这个新的`Model`作为该函数的输出——基本上`model()`函数调用我们`App`的业务逻辑（可以是交互、用例、`Repository`......您在`App`中使用的任何模式/术语）并作为结果提供新的`Model`对象。

### 3.view()

该方法获取`model()`函数返回的`Model`，并将其作为`view()`函数的输入，这之后通过某种方式将`Model`展示出来，`view()`和`view.render(model)`大体上是一致的。

### 4.本质

但是我们希望构建的是 **响应式的App**，不是吗？那么`MVI`是如何响应式的呢？响应式实际上意味着什么？

这意味着`App`的 **UI反映了状态的变更**。

因为`Model`反映了状态，因此，本质上我们希望 **业务逻辑能够对输入的事件（即`intents`）进行响应，并创建对应的`Model`作为输出，这之后再通过调用`View`层的`render(model)`方法，对UI进行渲染**。

### 5.通过RxJava串联

我们希望我们的数据流的单向性，因此`RxJava`闪亮登场。我们的App必须通过`RxJava`保持 **数据的单向性** 和 **响应式** 来构建吗？或者必须用`MVI`模式才能构建吗？当然不，我们也可以写 **命令式** 和 **程序性** 的代码。但是，**基于事件编程** 的`RxJava`实在太优秀了，既然`UI`是基于事件的，因此使用`RxJava`也是非常有意义的。

本文我们将会构建一个简单的虚拟在线商店`App`，其UI界面中展示的商品数据，都来源于我们向后台进行的网络请求。

我们可以精确的搜索特定的商品，并将其添加到我们的购物车中，最终`App`的效果如下所示：

![](https://upload-images.jianshu.io/upload_images/2583346-e832294ea5fc86aa.gif?imageMogr2/auto-orient/strip%7CimageView2/2/w/202/format/webp)

这个项目的源码你可以在[Github](https://github.com/sockeqwe/mosby/tree/master/sample-mvi)上找到，我们从实现一个简单的搜索界面开始做起：

首先，就像上文我们描述的那样，我们定义一个`Model`用于描述`View`层是如何被展示的—— **这个系列中，我们将用带有 `ViewState` 后缀的类来替代 `Model`**；举个例子，我们将会为搜索页的`Model`类命名为`SearchViewState`。

这很好理解，因为`Model`反应的就是状态（State），至于为什么不用听起来有些奇怪的名称比如`SearchModel`，是因为担心和`MVVM`中的`SearchViewModel`类在一起会导致歧义——命名真的很难。

```Java
public interface SearchViewState {

  // 搜索尚未开始
  final class SearchNotStartedYet implements SearchViewState {
  }

  // 搜索中
  final class Loading implements SearchViewState {
  }

  // 返回结果为空
  final class EmptyResult implements SearchViewState {
    private final String searchQueryText;

    public EmptyResult(String searchQueryText) {
      this.searchQueryText = searchQueryText;
    }

    public String getSearchQueryText() {
      return searchQueryText;
    }
  }

  // 有效的搜索结果，包含和搜索条件匹配的商品列表
  final class SearchResult implements SearchViewState {
    private final String searchQueryText;
    private final List<Product> result;

    public SearchResult(String searchQueryText, List<Product> result) {
      this.searchQueryText = searchQueryText;
      this.result = result;
    }

    public String getSearchQueryText() {
      return searchQueryText;
    }

    public List<Product> getResult() {
      return result;
    }
  }

  // 表示搜索过程中发生了错误
  final class Error implements SearchViewState {
    private final String searchQueryText;
    private final Throwable error;

    public Error(String searchQueryText, Throwable error) {
      this.searchQueryText = searchQueryText;
      this.error = error;
    }

    public String getSearchQueryText() {
      return searchQueryText;
    }

    public Throwable getError() {
      return error;
    }
  }
}
```

因为`Java`是一种强类型的语言，因此我们可以选择一种安全的方式为我们的`Model`类拆分出多个不同的 **子状态**。

我们的业务逻辑返回的是一个 **`SearchViewState`** 类型的对象，它可能是`SearchViewState.Error`或者其它的一个实例。这只是我个人的偏好，们也可以通过不同的方式定义，例如：

```Java
class SearchViewState {
  Throwable error;  // 非空则意味着，出现了一个错误
  boolean loading;  // 值为true意味着正在加载中
  List<Product> result; // 非空意味着商品列表的结果
  boolean SearchNotStartedYet; // true意味着还未开始搜索
}
```

再次重申，如何定义`Model`纯属个人喜好，如果你用`Kotlin`作为编程语言，那么`sealed classes`是一个不错的选择。

将目光聚集回到业务代码，让我们通过 **`SearchInteractor`** 去执行搜索的功能，其输出就是我们之前说过的`SearchViewState`对象：

```Java
public class SearchInteractor {
  final SearchEngine searchEngine; // 执行网络请求

  public Observable<SearchViewState> search(String searchString) {
    // 如果是空的字符串，不进行搜索
    if (searchString.isEmpty()) {
      return Observable.just(new SearchViewState.SearchNotStartedYet());
    }

    // 搜索商品列表
    // 返回 Observable<List<Product>>
    return searchEngine.searchFor(searchString)
        .map(products -> {
          if (products.isEmpty()) {
            return new SearchViewState.EmptyResult(searchString);
          } else {
            return new SearchViewState.SearchResult(searchString, products);
          }
        })
        .startWith(new SearchViewState.Loading())
        .onErrorReturn(error -> new SearchViewState.Error(searchString, error));
  }
}
```

来看下`SearchInteractor.search()`的方法签名：我们将`String`类型的`searchString`作为 **输入** 的参数，以及`Observable<SearchViewState>`类型的 **输出**，这意味着我们期望随着时间的推移，可以在可观察的流上会有任意多个`SearchViewState`的实例被发射。

在我们正式开始查询搜索之前（即`SearchEngine`执行网络请求），我们通过`startWith()`操作符发射一个`SearchViewState.Loading`,这将会使得`View`在执行搜索时展示`ProgressBar`。

`onErrorReturn()`会捕获在执行搜索时抛出的所有异常，并且发射出一个`SearchViewState.Error`——在订阅这个`Observable`时，我们为什么不去使用`onError()`回调呢？

这是一个对`RxJava`认知的普遍误解，实际上，`onError()`的回调意味着 **整个可观察的流进入了不可恢复的状态**，因此可观察的流结束了，而在我们的案例中，类似“没有网络连接”的`error`并非不可恢复的`error`：这只是我们的`Model`所代表的另外一个状态。

此外，我们还有另外一个可以转换到的状态，即一旦网络连接可用，我们可以通过 **`SearchViewState.Loading`** 跳转到的 **加载状态**。

因此，我们建立了一个可观察的流，这是一个每当状态发生了改变，从业务逻辑层就会发射一个发生了改变的`Model`到`View`层的流。

我们不想在网络连接错误时终止这个可观察的流，因此，在`error`发生时，类似这种可以被处理为 **状态** 的`error`（而不是终止流的那种致命的错误），可以反应为`Model`，被可观察的流发射。

通常，在`MVI`中，`Model`的`Observable`永远不会被终止（即永远不会执行`onComplete()`或者`onError()`回调）。

总结一下，`SearchInteractor`(即业务逻辑)提供了一个可观察的流`Observable<SearchViewState>`，每当状态发生了变化，就会发射一个新的`SearchViewState`。

### 6.View层的职责

接下来我们来讨论一下`View`应该是什么样的，`View`层的职责是什么？显然`View`层应该对`Model`进行展示，我们已经认可`View`层应该有类似 **`render(model)`** 这样的函数。此外，`View`应该提供一个给其他层响应用户输入的方法，在`MVI`中这个方法被称为 **`intents`**。

在这个案例中，我们只有一个`intent`:用户可以在输入框中输入一个用于检索商品的字符串进行搜索。`MVP`中的好习惯是为`View`层定义一个接口，所以在`MVI`中我们也可以这样做。

```Java
public interface SearchView {

  // 搜索的intent
  Observable<String> searchIntent();

  // 对View层进行渲染
  void render(SearchViewState viewState);
}
```

我们的案例中`View`层只提供了一个`intent`,但通常`View`拥有更多的`intent`;在 [第一小节](https://github.com/qingmei2/android-programming-profile/blob/master/src/Android-MVI/%5B%E8%AF%91%5D%E4%BD%BF%E7%94%A8MVI%E6%89%93%E9%80%A0%E5%93%8D%E5%BA%94%E5%BC%8FAPP%5B%E4%B8%80%5D%3AModel%E5%B1%82%E5%88%B0%E5%BA%95%E4%BB%A3%E8%A1%A8%E4%BB%80%E4%B9%88.md) 中我们讨论了为什么一个单独的`render()`函数是一个不错的实践，如果你对此还不是很清楚的话，请阅读该小节并通过留言进行探讨。

在我们开始对`View`层进行具体的实现之前，我们先看看最终界面的展示效果：

![](https://upload-images.jianshu.io/upload_images/2583346-55cc5ec652fa3e40.gif?imageMogr2/auto-orient/strip%7CimageView2/2/w/398/format/webp)

```Java
public class SearchFragment extends Fragment implements SearchView {

  @BindView(R.id.searchView) android.widget.SearchView searchView;
  @BindView(R.id.container) ViewGroup container;
  @BindView(R.id.loadingView) View loadingView;
  @BindView(R.id.errorView) TextView errorView;
  @BindView(R.id.recyclerView) RecyclerView recyclerView;
  @BindView(R.id.emptyView) View emptyView;
  private SearchAdapter adapter;

  @Override public Observable<String> searchIntent() {
    return RxSearchView.queryTextChanges(searchView) // 感谢 Jake Wharton :)
        .filter(queryString -> queryString.length() > 3 || queryString.length() == 0)
        .debounce(500, TimeUnit.MILLISECONDS);
  }

  @Override public void render(SearchViewState viewState) {
    if (viewState instanceof SearchViewState.SearchNotStartedYet) {
      renderSearchNotStarted();
    } else if (viewState instanceof SearchViewState.Loading) {
      renderLoading();
    } else if (viewState instanceof SearchViewState.SearchResult) {
      renderResult(((SearchViewState.SearchResult) viewState).getResult());
    } else if (viewState instanceof SearchViewState.EmptyResult) {
      renderEmptyResult();
    } else if (viewState instanceof SearchViewState.Error) {
      renderError();
    } else {
      throw new IllegalArgumentException("Don't know how to render viewState " + viewState);
    }
  }

  private void renderResult(List<Product> result) {
    TransitionManager.beginDelayedTransition(container);
    recyclerView.setVisibility(View.VISIBLE);
    loadingView.setVisibility(View.GONE);
    emptyView.setVisibility(View.GONE);
    errorView.setVisibility(View.GONE);
    adapter.setProducts(result);
    adapter.notifyDataSetChanged();
  }

  private void renderSearchNotStarted() {
    TransitionManager.beginDelayedTransition(container);
    recyclerView.setVisibility(View.GONE);
    loadingView.setVisibility(View.GONE);
    errorView.setVisibility(View.GONE);
    emptyView.setVisibility(View.GONE);
  }

  private void renderLoading() {
    TransitionManager.beginDelayedTransition(container);
    recyclerView.setVisibility(View.GONE);
    loadingView.setVisibility(View.VISIBLE);
    errorView.setVisibility(View.GONE);
    emptyView.setVisibility(View.GONE);
  }

  private void renderError() {
    TransitionManager.beginDelayedTransition(container);
    recyclerView.setVisibility(View.GONE);
    loadingView.setVisibility(View.GONE);
    errorView.setVisibility(View.VISIBLE);
    emptyView.setVisibility(View.GONE);
  }

  private void renderEmptyResult() {
    TransitionManager.beginDelayedTransition(container);
    recyclerView.setVisibility(View.GONE);
    loadingView.setVisibility(View.GONE);
    errorView.setVisibility(View.GONE);
    emptyView.setVisibility(View.VISIBLE);
  }
}
```

`render(SearchViewState)`方法的作用显而易见，`searchIntent()`方法中，我们使用了`Jake Wharton`的 [RxBinding](https://github.com/JakeWharton/RxBinding) ，这是一个对`Android UI`组件提供了`RxJava`响应式支持的库。

`RxSearchView.queryText()`创建了一个`Observable<String>`,每当用户在`EditText`上输入了一些文字，它就会发射一个对应的字符串；我们通过`filter()`去保证只有当用户输入的字符数达到三个以上时才进行搜索；同时，我们不希望每当用户输入一个字符，就去请求网络，而是当用户输入结束后再去请求网络（`debounce()`操作符会停留500毫秒以决定用户是否输入完成）。

现在我们知道了屏幕中的`searchIntent()`方法就是 **输入** ，而`render()`方法则是 **输出**。我们如何从 **输入** 获得 **输出** 呢，如下所示：

![](https://upload-images.jianshu.io/upload_images/2583346-4289ff885fd27824.gif?imageMogr2/auto-orient/strip%7CimageView2/2/w/1000/format/webp)

### 7.连接View和Intent

剩下的问题就是：我们如何将`View`的`intent`和业务逻辑进行连接呢？如果你认真观看了上面的流程图，你应该注意到了中间的 **`flatMap()`** 操作符，这暗示了我们还有一个尚未谈及的组件： **`Presenter`** ;`Presenter`负责连接这些点，就和我们在`MVP`中使用的方式一样。

```Java
public class SearchPresenter extends MviBasePresenter<SearchView, SearchViewState> {
  private final SearchInteractor searchInteractor;

  @Override protected void bindIntents() {
    Observable<SearchViewState> search =
        intent(SearchView::searchIntent)
        // 上文中我们谈到了flatMap,但在这里switchMap更为适用
            .switchMap(searchInteractor::search)
            .observeOn(AndroidSchedulers.mainThread());

    subscribeViewState(search, SearchView::render);
  }
}
```

什么是 **`MviBasePresenter`**, **`intent()`** 和 **`subscribeViewState()`** 又是什么？这个类是我写的 [Mosby](https://github.com/sockeqwe/mosby) 库的一部分（3.0版本后，`Mosby`已经支持了`MVI`）。本文并非为了讲述`Mosby`，但我向简单介绍一下`MviBasePresenter`是如何的便利——这其中没有什么黑魔法，虽然确实看起来像是那样。

让我们从生命周期开始：`MviBasePresenter`并未持有任何生命周期，它暴露出一个 **`bindIntent()`** 方法以供`View`层和业务逻辑进行绑定。通常，你通过`flatMap()`、`switchMap()`或者`concatMap()`操作符将`intent` “转移”到业务逻辑中，这个方法仅仅在`View`层第一次被附加到`Presenter`中时调用，而当`View`再次被附加在`Presenter`中时（比如，屏幕方向发生了改变），将不再被调用。

这听起来有些怪，也许有人会说：

> `MviBasePresenter`在屏幕方向发生了改变后依然能够存活？如果是这样，`Mosby`如何保证`Observable`的流不会发生内存的泄漏？

这就是 **`intent()`** 和 **`subscribeViewState()`** 的作用所在了，**`intent()`** 在内部创建一个`PublishSubject`，就像是业务逻辑的“网关”一样；实际上,`PublishSubject`订阅了`View`层传过来的`intent`的`Observable`,调用`intent(o1)`实际返回了一个订阅了`o1`的`PublishSubject`。

屏幕发生旋转时，`Mosby`将`View`从`Presenter`中分离，但是，内部的`PublishSubject`只是暂时和`View`解除了订阅；而当`View`重新附着在`Presenter`上时，`PublishSubject`将会对`View`层的`intent`进行重新订阅。

`subscribeViewState()`方法做的是同样的事情，只不过将顺序调换了过来（`Presenter`向`View`层的通信）。它在内部创建一个`BehaviorSubject`作为从业务逻辑到View层的“网关”。

由于它是一个`BehaviorSubject`，因此，即使此时`Presenter`没有持有`View`，我们依然可以从业务逻辑中接收到`Model`的更新（比如`View`并未处于栈顶）；`BehaviorSubjects`始终持有它最后的值，并在`View`重新依附后将其重新发射。

规则很简单：使用`intent()`来“包装”`View`层的所有`intent`,使用`subscribeViewState()`替代`Observable.subscribe()`.

![](http://hannesdorfmann.com/images/mvi-mosby3/MviBasePresenter.png)

### 8.UnbindIntents

与`bindIntent()`相对应的是 **`unbindIntents()`** ,该方法只会执行一次，即`View`被永久销毁时才会被调用。举个例子，将一个`Fragment`放在栈中，直到`Activity`被销毁之前，该`View`一直不会被销毁。

由于`intent()`和`subscribeViewState()`已经对订阅进行了管理，因此您只需要实现`unbindIntents()`。

### 9.其它生命周期的事件

那么其它生命周期的事件，比如`onPause()`和`onResume()`又该如何处理？我依然认为`Presenter`不需要生命周期的事件，然而，如果你坚持认为你需要将这些生命周期的事件视为另一种形式的`intent`，您的`View`可以提供一个`pauseIntent()`，它是由`android`生命周期触发，而又不是按钮点击事件这样的由用户交互触发的`intent`——但两者都是有效的意图。

## 结语

第二小节中，我们探讨了`Model-View-Intent`的基础，并通过`MVI`浅尝辄止实现了一个简单的页面。也许这个例子太简单了，所以你尚未感受到`MVI`模式的优点：代表 **状态** 的`Model`和与传统`MVP`或者`MVVM`相比的 **单项数据流**。

`MVP`和`MVVM`并没有什么问题，我也并非是在说`MVI`比其它架构模式更优秀，但是，我认为`MVI`可以帮助我们 **为复杂的问题编写优雅的代码** ，这也正如我们将在本系列博客的 [下一小节](https://github.com/qingmei2/android-programming-profile/blob/master/src/Android-MVI/%5B%E8%AF%91%5D%E4%BD%BF%E7%94%A8MVI%E6%89%93%E9%80%A0%E5%93%8D%E5%BA%94%E5%BC%8FAPP%5B%E4%B8%89%5D%3AStateReducer.md)（第3小节）中探讨的那样——届时我们将针对 **状态折叠器** （state reducers）的问题进行探讨，欢迎关注。


**--------------------------广告分割线------------------------------**

**《使用MVI打造响应式APP》翻译系列**  
> * [[译]使用MVI打造响应式APP(一):Model到底是什么](https://github.com/qingmei2/android-programming-profile/blob/master/src/Android-MVI/%5B%E8%AF%91%5D%E4%BD%BF%E7%94%A8MVI%E6%89%93%E9%80%A0%E5%93%8D%E5%BA%94%E5%BC%8FAPP%5B%E4%B8%80%5D%3AModel%E5%B1%82%E5%88%B0%E5%BA%95%E4%BB%A3%E8%A1%A8%E4%BB%80%E4%B9%88.md)  
> * [[译]使用MVI打造响应式APP[二]:View层和Intent层](https://github.com/qingmei2/android-programming-profile/blob/master/src/Android-MVI/%5B%E8%AF%91%5D%E4%BD%BF%E7%94%A8MVI%E6%89%93%E9%80%A0%E5%93%8D%E5%BA%94%E5%BC%8FAPP%5B%E4%BA%8C%5D%3AView%E5%B1%82%E5%92%8CIntent%E5%B1%82.md)  
> * [[译]使用MVI打造响应式APP[三]:状态折叠器](https://github.com/qingmei2/android-programming-profile/blob/master/src/Android-MVI/%5B%E8%AF%91%5D%E4%BD%BF%E7%94%A8MVI%E6%89%93%E9%80%A0%E5%93%8D%E5%BA%94%E5%BC%8FAPP%5B%E4%B8%89%5D%3AStateReducer.md)  
> * [[译]使用MVI打造响应式APP[四]:IndependentUIComponents](https://github.com/qingmei2/android-programming-profile/blob/master/src/Android-MVI/%5B%E8%AF%91%5D%E4%BD%BF%E7%94%A8MVI%E6%89%93%E9%80%A0%E5%93%8D%E5%BA%94%E5%BC%8FAPP%5B%E5%9B%9B%5D%3AIndependentUIComponents.md)  
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
