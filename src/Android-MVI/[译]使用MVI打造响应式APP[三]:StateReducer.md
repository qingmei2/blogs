# [译]使用MVI打造响应式APP(三):状态折叠器

> 原文：[REACTIVE APPS WITH MODEL-VIEW-INTENT - PART3 - STATE REDUCER](http://hannesdorfmann.com/android/mosby3-mvi-3)  
作者：[Hannes Dorfmann](http://hannesdorfmann.com)  
译者：[却把清梅嗅](https://github.com/qingmei2)

在[上一章节](https://github.com/qingmei2/android-programming-profile/blob/master/src/Android-MVI/%5B%E8%AF%91%5D%E4%BD%BF%E7%94%A8MVI%E6%89%93%E9%80%A0%E5%93%8D%E5%BA%94%E5%BC%8FAPP%5B%E4%BA%8C%5D%3AView%E5%B1%82%E5%92%8CIntent%E5%B1%82.md)中,我们针对 **如何使用单向流和 Model-View-Intent 模式构建一个简单的页面** 进行了探讨；本章节，我们将在`reducer`的帮助下实现`MVI`模式中更加复杂的页面。

如果你还未阅读前两个章节，阅读本文之前您应该先去阅读它们，从而对如下两个问题的答案有初步的了解：

* 1.我们如何通过`Presenter`将`View`层和业务逻辑相关联？
* 2.数据流是如何保证单向性的？

如下图所示，现在我们构建这样一个复杂的页面：

如你所见，屏幕中显示的是按照类别进行归类的商品列表；`App`每次只会为每个分类展示3个条目，当用户点击了 **加载更多** 按钮时，将会通过网络请求去加载该分类下所有的条目。

此外，用户还可以执行 **下拉刷新** 的操作，并且一旦用户向下滚动到列表末尾，分页功能就会继续加载下一页的数据——当然，所有这些行为可以同时执行，并且每个行为都可能会收到失败（即没有互联网连接）。

让我们一步步来，首先，我们先对`View`层的接口进行实现：

```Java
public interface HomeView {

  /**
   * 加载第一页数据的intent
   *
   * @return 发射的数据是没有意义的，true或者false没有区别
   */
  public Observable<Boolean> loadFirstPageIntent();

  /**
   * 分页加载下一页的intent
   *
   * @return 发射的数据是没有意义的，true或者false没有区别
   */
  public Observable<Boolean> loadNextPageIntent();

  /**
   * 对下拉刷新的响应intent
   *
   * @return 发射的数据是没有意义的，true或者false没有区别
   */
  public Observable<Boolean> pullToRefreshIntent();

  /**
   * 根据当前分类加载所有条目的intent
   *
   * @return 指定分类，String代表分类的名字
   */
  public Observable<String> loadAllProductsFromCategoryIntent();

  /**
   * 对ViewState进行渲染
   */
  public void render(HomeViewState viewState);
}
```

`View`层具体的实现简单明了，本文将不进行展示（但你可以在[Github](https://github.com/sockeqwe/mosby/blob/master/sample-mvi/src/main/java/com/hannesdorfmann/mosby3/sample/mvi/view/home/HomeFragment.java)上找到它）。

接下来让我们把目光转向`Model`，正如前文所提到的，**`Model`应该反应了状态**，现在我来介绍一下`Model`的具体实现：**HomeViewState**。

```java
public final class HomeViewState {

  private final boolean loadingFirstPage; // RecyclerView加载状态的指示器
  private final Throwable firstPageError; // 如果非空，展示一个error
  private final List<FeedItem> data;   // 列表的数据
  private final boolean loadingNextPage; // RecyclerView分页加载状态的指示器
  private final Throwable nextPageError; // 如果非空，展示分页error的toast
  private final boolean loadingPullToRefresh; // 展示下拉刷新状态的指示器
  private final Throwable pullToRefreshError; // 非空意味着下拉刷新的error

   // ... 构造器 ...
   // ... getter方法  ...
}
```

请注意，**FeedItem** 仅仅是一个接口，每个条目都需要实现该接口，然后交给`RecyclerView`去展示。比如 **Product** 实现了 **FeedItem**；此外，列表中的类别标题 **SectionHeader** 也实现了 **FeedItem**；还有，作为UI中的元素之一，表示 “可以加载该类别更多” 的指示器同样也是 **FeedItem**，其内部还持有了一个小状态——该状态代表了当前是否 **正在加载更多条目** 。

```Java
public class AdditionalItemsLoadable implements FeedItem {
  private final int moreItemsAvailableCount;
  private final String categoryName;
  private final boolean loading; // true 代表item正处于加载状态
  private final Throwable loadingError; // 标志loading时捕获到了error

   // ... 构造器 ...
   // ... getter方法  ...
```

这之后便是压轴的业务逻辑组件 **HomeFeedLoader** ，它负责对 **FeedItems** 进行加载：

```java
public class HomeFeedLoader {

  // 通常由 下拉刷新 动作触发
  public Observable<List<FeedItem>> loadNewestPage() { ... }

  // 加载第一页
  public Observable<List<FeedItem>> loadFirstPage() { ... }

  // 加载下一页
  public Observable<List<FeedItem>> loadNextPage() { ... }

  // 加载某个分类的其它产品
  public Observable<List<Product>> loadProductsOfCategory(String categoryName) { ... }
}
```

现在，让我们一步步将这些点在`Presenter`中进行连接。请注意，接下来`Presenter`中展示的部分代码，在真实的开发中，应该被转移到`Interactor`(交互器)中（这并非是为了更好的可读性）。首先，我们先开始对初始化数据进行加载：

```Java
class HomePresenter extends MviBasePresenter<HomeView, HomeViewState> {

  private final HomeFeedLoader feedLoader;

  @Override protected void bindIntents() {

    // 在真实的开发中，应该被转移到Interactor中
    Observable<HomeViewState> loadFirstPage = intent(HomeView::loadFirstPageIntent)
        .flatMap(ignored -> feedLoader.loadFirstPage()
            .map(items -> new HomeViewState(items, false, null) )
            .startWith(new HomeViewState(emptyList, true, null) )
            .onErrorReturn(error -> new HomeViewState(emptyList, false, error))

    subscribeViewState(loadFirstPage, HomeView::render);
  }
}
```

到目前为止感觉良好，和[上一章节](https://github.com/qingmei2/android-programming-profile/blob/master/src/Android-MVI/%5B%E8%AF%91%5D%E4%BD%BF%E7%94%A8MVI%E6%89%93%E9%80%A0%E5%93%8D%E5%BA%94%E5%BC%8FAPP%5B%E4%BA%8C%5D%3AView%E5%B1%82%E5%92%8CIntent%E5%B1%82.md)我们实现的`Search`界面相比，没有什么太大的不同。

现在我们尝试添加对 **下拉刷新** 的支持：、

```Java
class HomePresenter extends MviBasePresenter<HomeView, HomeViewState> {

  private final HomeFeedLoader feedLoader;

  @Override protected void bindIntents() {

    // 在真实的开发中，应该被转移到Interactor中
    Observable<HomeViewState> loadFirstPage = ... ;

    Observable<HomeViewState> pullToRefresh = intent(HomeView::pullToRefreshIntent)
        .flatMap(ignored -> feedLoader.loadNewestPage()
            .map( items -> new HomeViewState(...))
            .startWith(new HomeViewState(...))
            .onErrorReturn(error -> new HomeViewState(...)));

    Observable<HomeViewState> allIntents = Observable.merge(loadFirstPage, pullToRefresh);

    subscribeViewState(allIntents, HomeView::render);
  }
}
```

稍微等一下：**`feedLoader.loadNewestPage()`** 仅仅返回了新的条目数据，**但是之前我们已经加载了的条目怎么办**？

“传统”的`MVP`模式中，我们可以调用类似`view.addNewItems(newItems)`的方法，但是在 [第一篇文章](https://github.com/qingmei2/android-programming-profile/blob/master/src/Android-MVI/%5B%E8%AF%91%5D%E4%BD%BF%E7%94%A8MVI%E6%89%93%E9%80%A0%E5%93%8D%E5%BA%94%E5%BC%8FAPP%5B%E4%B8%80%5D%3AModel%E5%B1%82%E5%88%B0%E5%BA%95%E4%BB%A3%E8%A1%A8%E4%BB%80%E4%B9%88.md) 中，我们已经探讨了为什么这不是一个好主意（状态问题）。

我们当前面临的问题是，**下拉刷新依赖了之前的状态**，因为我们想要将下拉刷新返回的条目和之前已经加载的条目进行 **合并**。

**女士们，先生们，现在，让我们热情地欢迎状态折叠器（State Reducer）的到来！**

![](http://hannesdorfmann.com/images/mvi-mosby3/standingovation3.gif)

`State Reducer`是函数式编程中的一个概念，它 **将前一个状态作为输入，并根据前一个状态计算得出一个新的状态**，就像这样：

```Java
public State reduce( State previous, Foo foo ){
  State newState;
  // ... 根据前一个状态计算得出一个新的状态 ...
  return newState;
}
```

因此上述问题的解决方案是，我们定义一个`Foo`组件，通过其类似`reduce()`的函数，结合之前的状态计算出一个新的状态。

这个名为`Foo`的组件通常意味着我们希望对**之前状态所进行的改变**，在我们的案例中，我们希望将 **最初通过`loadFirstPageIntent`计算得到的`HomeViewState`** 和 **下拉刷新得到的结果** 进行`reduce`。

你猜怎么着，`RxJava`有一个名为 **`scan()`** 的操作符，让我们对我们的代码进行略微的重构，我们需要引入另外一个表示 **部分改变** 的类—— 上面我们将其称之为`Foo`，它将用于计算新的状态。

```java
class HomePresenter extends MviBasePresenter<HomeView, HomeViewState> {

  private final HomeFeedLoader feedLoader;

  @Override protected void bindIntents() {

    Observable<PartialState> loadFirstPage = intent(HomeView::loadFirstPageIntent)
        .flatMap(ignored -> feedLoader.loadFirstPage()
            .map(items -> new PartialState.FirstPageData(items) )
            .startWith(new PartialState.FirstPageLoading(true) )
            .onErrorReturn(error -> new PartialState.FirstPageError(error))

    Observable<PartialState> pullToRefresh = intent(HomeView::pullToRefreshIntent)
        .flatMap(ignored -> feedLoader.loadNewestPage()
            .map( items -> new PartialState.PullToRefreshData(items)
            .startWith(new PartialState.PullToRefreshLoading(true)))
            .onErrorReturn(error -> new PartialState.PullToRefreshError(error)));

    Observable<PartialState> allIntents = Observable.merge(loadFirstPage, pullToRefresh);
    // 展示第一页数据加载中...
    HomeViewState initialState = ... ;
    Observable<HomeViewState> stateObservable = allIntents.scan(initialState, this::viewStateReducer)

    subscribeViewState(stateObservable, HomeView::render);
  }

  private HomeViewState viewStateReducer(HomeViewState previousState, PartialState changes){
     ...
   }
 }

```

我们在这里做了什么？相比较直接返回`Observable<HomeViewState>`，现在每个`Intent`返回的是`Observable<PartialState>`。这之后我们通过`merge()`操作符将其全部合并为一个可观察的流中，并最终应用到了`reducer`的函数中（即`Observable.scan()`）。

这意味着，无论何时用户发起了一个`intent`,这个`intent`将会生产一个`PartialState`的实例，然后被`reduced`得到了`HomeViewState`,最终，被`View`层进行展示（`HomeView.render(HomeViewState)`）。

唯一遗漏的部分应该就是`state reducer`的函数本身了，如上文中的定义一样，`HomeViewState`类本身并未发生了改变，但是我们通过`Builder`模式添加了一个`Builder`，这样我们就可以非常便捷地创建一个新的`HomeViewState`实例。

现在让我们开始实现`state reducer`的函数：

```java
private HomeViewState viewStateReducer(HomeViewState previousState, PartialState changes){
    if (changes instanceof PartialState.FirstPageLoading)
        return previousState.toBuilder() // 根据当前状态复制一个内部同样状态的对象
        .firstPageLoading(true) // 展示progressBar
        .firstPageError(null) // 不展示error
        .build()

    if (changes instanceof PartialState.FirstPageError)
     return previousState.builder()
         .firstPageLoading(false) // 隐藏progressBar
         .firstPageError(((PartialState.FirstPageError) changes).getError()) // 展示error
         .build();

     if (changes instanceof PartialState.FirstPageLoaded)
       return previousState.builder()
           .firstPageLoading(false)
           .firstPageError(null)
           .data(((PartialState.FirstPageLoaded) changes).getData())
           .build();

     if (changes instanceof PartialState.PullToRefreshLoading)
      return previousState.builder()
            .pullToRefreshLoading(true) // 展示下拉刷新的UI指示器
            .nextPageError(null)
            .build();

    if (changes instanceof PartialState.PullToRefreshError)
      return previousState.builder()
          .pullToRefreshLoading(false) // 隐藏下拉刷新的UI指示器
          .pullToRefreshError(((PartialState.PullToRefreshError) changes).getError())
          .build();

    if (changes instanceof PartialState.PullToRefreshData) {
      List<FeedItem> data = new ArrayList<>();
      data.addAll(((PullToRefreshData) changes).getData()); // 将新的数据插入到当前列表的顶部
      data.addAll(previousState.getData());
      return previousState.builder()
        .pullToRefreshLoading(false)
        .pullToRefreshError(null)
        .data(data)
        .build();
    }


   throw new IllegalStateException("Don't know how to reduce the partial state " + changes);
}
```

我知道，这些代码看起来并不优雅，但这不是本文的重点——为什么博主会在他的文章中展示如此 **“丑陋”** 的代码？

因为我希望能够阐述一个观点，我认为 **读者并不应该为源码中错综复杂的逻辑买单** ，比如，我们的购物车`App`中，也不需要读者对某些设计模式有额外的知识储备。

因此，我认为博客文章中最好避免出现设计模式，这的确会展示出更好的代码，但其本身就意味着 **更高的阅读理解成本**。

回顾本文，其重点是对`State Reducer`进行配置，通过上述的代码，大家都能够更快更准确地去了解它是什么。但你会在实际开发中这样编写代码吗？当然不会，我会去使用设计模式或者其它的解决方案，比如使用 **`public HomeViewState computeNewState(previousState)`** 之类的方法将`PartialState`定义为接口。

好吧，我想你已经了解了`State Reducer`是如何工作的，让我们实现剩下来的功能：分页以及能够加载某个指定分类更多的Item：

```Java
class HomePresenter extends MviBasePresenter<HomeView, HomeViewState> {

  private final HomeFeedLoader feedLoader;

  @Override protected void bindIntents() {

    Observable<PartialState> loadFirstPage = ... ;
    Observable<PartialState> pullToRefresh = ... ;

    Observable<PartialState> nextPage =
      intent(HomeView::loadNextPageIntent)
          .flatMap(ignored -> feedLoader.loadNextPage()
              .map(items -> new PartialState.NextPageLoaded(items))
              .startWith(new PartialState.NextPageLoading())
              .onErrorReturn(PartialState.NexPageLoadingError::new));

      Observable<PartialState> loadMoreFromCategory =
          intent(HomeView::loadAllProductsFromCategoryIntent)
              .flatMap(categoryName -> feedLoader.loadProductsOfCategory(categoryName)
                  .map( products -> new PartialState.ProductsOfCategoryLoaded(categoryName, products))
                  .startWith(new PartialState.ProductsOfCategoryLoading(categoryName))
                  .onErrorReturn(error -> new PartialState.ProductsOfCategoryError(categoryName, error)));


    Observable<PartialState> allIntents = Observable.merge(loadFirstPage, pullToRefresh, nextPage, loadMoreFromCategory);
    // 展示第一页正在加载
    HomeViewState initialState = ... ;
    Observable<HomeViewState> stateObservable = allIntents.scan(initialState, this::viewStateReducer)

    subscribeViewState(stateObservable, HomeView::render);
  }

  private HomeViewState viewStateReducer(HomeViewState previousState, PartialState changes){
    // ... 第一页的部分状态处理和下拉刷新 ...

      if (changes instanceof PartialState.NextPageLoading) {
       return previousState.builder().nextPageLoading(true).nextPageError(null).build();
     }

     if (changes instanceof PartialState.NexPageLoadingError)
       return previousState.builder()
           .nextPageLoading(false)
           .nextPageError(((PartialState.NexPageLoadingError) changes).getError())
           .build();


     if (changes instanceof PartialState.NextPageLoaded) {
       List<FeedItem> data = new ArrayList<>();
       data.addAll(previousState.getData());
        // 将新的数据添加到list的尾部
       data.addAll(((PartialState.NextPageLoaded) changes).getData());

       return previousState.builder().nextPageLoading(false).nextPageError(null).data(data).build();
     }

     if (changes instanceof PartialState.ProductsOfCategoryLoading) {
         int indexLoadMoreItem = findAdditionalItems(categoryName, previousState.getData());

         AdditionalItemsLoadable ail = (AdditionalItemsLoadable) previousState.getData().get(indexLoadMoreItem);

         AdditionalItemsLoadable itemsThatIndicatesError = ail.builder() // 创建所有item的副本
         .loading(true).error(null).build();

         List<FeedItem> data = new ArrayList<>();
         data.addAll(previousState.getData());
         data.set(indexLoadMoreItem, itemsThatIndicatesError); // 这将会展示一个loading的指示器

         return previousState.builder().data(data).build();
      }

     if (changes instanceof PartialState.ProductsOfCategoryLoadingError) {
       int indexLoadMoreItem = findAdditionalItems(categoryName, previousState.getData());

       AdditionalItemsLoadable ail = (AdditionalItemsLoadable) previousState.getData().get(indexLoadMoreItem);

       AdditionalItemsLoadable itemsThatIndicatesError = ail.builder().loading(false).error( ((ProductsOfCategoryLoadingError)changes).getError()).build();

       List<FeedItem> data = new ArrayList<>();
       data.addAll(previousState.getData());
       data.set(indexLoadMoreItem, itemsThatIndicatesError); // 这将会展示一个error和重试的button
       return previousState.builder().data(data).build();
     }

     if (changes instanceof PartialState.ProductsOfCategoryLoaded) {
       String categoryName = (ProductsOfCategoryLoaded) changes.getCategoryName();
       int indexLoadMoreItem = findAdditionalItems(categoryName, previousState.getData());
       int indexOfSectionHeader = findSectionHeader(categoryName, previousState.getData());

       List<FeedItem> data = new ArrayList<>();
       data.addAll(previousState.getData());
       removeItems(data, indexOfSectionHeader, indexLoadMoreItem); // 移除指定分类下的所有item

       // 添加指定分类下的所有item (包括之前已经被移除的)
       data.addAll(indexOfSectionHeader + 1,((ProductsOfCategoryLoaded) changes).getData());

       return previousState.builder().data(data).build();
     }

     throw new IllegalStateException("Don't know how to reduce the partial state " + changes);
  }
}
```

实现分页加载和下拉刷新十分相似，异同之处仅仅在于前者是把加载到的数据添加在列表末尾，而下拉刷新则是把数据展示在界面顶部。

更有趣的是我们如何针对某个类别去加载更多条目：为了展示某个类别的加载指示器和错误/重试的按钮，我们只需在所有的`FeedItems`列表中找到对应的`AdditionalItemsLoadable`对象，然后我们将其改变为展示加载指示器或者错误/重试的按钮。

如果我们已成功加载某个类别的所有条目，我们将搜索`SectionHeader`和`AdditionalItemsLoadable`，并用新加载的列表替换这里的所有条目，仅此而已。

## 结语

本文的目的是向您展示 **状态折叠器（State Reducer）** 如何帮助我们通过 **简洁且易读** 的代码构建复杂的页面。现在回过头来思考，“传统”的`MVP`或者`MVVM`针对这些功能，在不使用`State Reducer`的前提下是如何实现这些功能的。

显然，能够使用`State Reducer`的关键是我们有一个反映状态的`Model`类，这也印证了该系列的第一篇文章中所阐述的，为什么理解 **Model** 是那么的重要。

此外，只有当我们确定状态（或准确的`Model`）来自单一的数据源时，才能使用`State Reducer`，因此单向数据流同样非常重要。

我希望我们花费在 **阅读** 并 **理解** 前两篇博客的时间是有意义的，现在，所有的点都成功的连在了一起，是时候欢呼了。

如果还没有，不用担心，对此我也花了相当长的时间才完全理解——还有很多次练习、错误和重试。

在第二篇博客中，针对搜索界面，我们并未使用`State Reducer`。这是因为如果我们以某种方式依赖于先前的状态，`State Reducer`是有意义的。而在“搜索界面”中，我们不依赖于先前的状态。

虽然在最后，但是我还是想重申，也许你还没有注意到，那就是我们的`data`都是不可变的——我们总是创建`HomeViewState`新的实例，而不是在已有的对象上调用其`setter`方法，这也使得多线程不再是问题。

用户可以在加载下一页的同时开始下拉刷新并加载某个类别的更多条目，因为`State Reducer`总是能够产生正确的状态，却不依赖于http响应的任何特定顺序。另外，我们用纯函数编写了代码，没有任何副作用。这使我们的代码非常具有可测试性、可重现性、易于推演和高度可并行化（即多线程）。

当然，`State Reducer`并非是`MVI`发明的，您可以在多种编程语言的许多三方库，框架和系统中找到其概念。它完全符合`Model-View-Intent`的理念，具有单向的数据流和表示状态的`Model`。

在下一个部分中，我们将聚焦于如何通过`MVI` 构建 **可复用** 和 **响应式** 的UI组件，敬请关注。


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
