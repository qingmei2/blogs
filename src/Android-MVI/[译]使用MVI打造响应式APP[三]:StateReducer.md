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

> 译者注：在 **函数式编程** 或 **Redux** 等领域中，`reducer`（译为减速器、缩减器、折叠器等）作为专业术语一般不对其翻译，下文同。
