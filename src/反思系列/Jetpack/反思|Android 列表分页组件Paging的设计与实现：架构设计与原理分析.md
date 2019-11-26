# 反思|Android 列表分页组件Paging的设计与实现：架构设计与原理分析

本文是`Android Jetpack Paging`系列的第二篇文章；**强烈建议** 读者将本系列作为学习`Paging` **阅读优先级最高的文章**，如果读者对`Paging`还没有系统性的认识，请参考：

* [反思|Android 列表分页组件Paging的设计与实现：系统概述](https://github.com/qingmei2/blogs/issues/30)

## 前言

`Paging`是一个非常优秀的分页组件，与其它热门的分页相关库不同的是，`Paging`更偏向注重服务于 **业务** 而非 **UI** 。——我们都知道业务类型的开源库的质量非常依赖代码 **整体的架构设计**（比如`Retofit`和`OkHttp`）；那么，如何说服自己或者同事去尝试使用`Paging`？显然源码中蕴含的优秀思想更具有说服力。

反过来说，若从`Google`工程师们设计、研发和维护的源码中有所借鉴，即使不在项目中真正使用它，自己依然能受益匪浅。

## 一、创建流程设计

### 1、经典案例：通过建造者模式进行依赖注入

创建流程毫无疑问是架构设计中最重要的环节。

作为组件的门板，向外暴露的`API`对于开发者越简单友善方便调用越好，同时，作为`API`调用者的我们也希望框架越灵活，可配置选项越多越好。

这听起来似乎有点违反常理—— 如何才能保证既保证 **简单干净的接口设计** 易于开发者上手，同时又有 **足够多的可配置项** 保证框架的灵活呢？

`Paging`的`API`设计中使用了经典的 **建造者（Builder）模式**，并通过依赖注入将依赖一层层向下传递，最终依次构建了各个层级的对象实例。

对于开发者而言，只需要配置自己关心的参数，而不关心（甚至可以是不知道）的参数配置，全交给`Builder`类使用默认参数：

```Kotlin
// 你可以这样复杂地配置
val pagedListLiveData =
    LivePagedListBuilder(
            dataSourceFactory,
            PagedList.Config.Builder()
                    .setPageSize(PAGE_SIZE)                         // 分页加载的数量
                    .setInitialLoadSizeHint(20)                     // 初始化加载的数量
                    .setPrefetchDistance(10)                        // 预加载距离
                    .setEnablePlaceholders(ENABLE_PLACEHOLDERS)     // 是否启用占位符
                    .build()
    ).build()

// 也可以这样简单地配置
val pagedListLiveData =
    LivePagedListBuilder(dataSourceFactory, PAGE_SIZE).build()
```

需要注意的是，**分页相关功能配置对象的构建** 和 **可观察者对象的构建** 是否是两个不同的职责？显然是有必要的，因为：

> `LiveData<PagedList>` = `DataSource` + `PagedList.Config`（即 分页数据的可观察者 = 数据源 + 分页配置）

因此，这里`Paging`的配置使用到了2个`Builder`类，即使是决定使用 **建造者模式** ，设计者也需要对`Builder`类的定义有一个清晰的认知，这里也是设计过程中 **单一职责原则** 的优秀体现。

最终，`Builder`中的所有配置都通过依赖注入的方式对`PagedList`进行了实例化：

```Java
// PagedList.Builder.build()
public PagedList<Value> build() {
    return PagedList.create(
            mDataSource,
            mNotifyExecutor,
            mFetchExecutor,
            mBoundaryCallback,
            mConfig,
            mInitialKey);
}

// PagedList.create()
static <K, T> PagedList<T> create(@NonNull DataSource<K, T> dataSource,
            @NonNull Executor notifyExecutor,
            @NonNull Executor fetchExecutor,
            @Nullable BoundaryCallback<T> boundaryCallback,
            @NonNull Config config,
            @Nullable K key) {
    // 这里我们仅以ContiguousPagedList为例
    // 可以看到，所有PagedList都是将构造函数的依赖注入进行的实例化
    return new ContiguousPagedList<>(contigDataSource,
          notifyExecutor,
          fetchExecutor,
          boundaryCallback,
          config,
          key,
          lastLoad);
}
```

**依赖注入** 是一个非常简单而又朴实的编码技巧，`Paging`的设计中，几乎没有用到单例模式，也几乎没有太多的静态成员——所有对象中除了自身的状态，其它所有通过依赖注入的配置项都是 **final** (不可变)的:

```Java
// PagedList.java
public abstract class PagedList<T> {
  final Executor mMainThreadExecutor;
  final Executor mBackgroundThreadExecutor;
  final BoundaryCallback<T> mBoundaryCallback;
  final Config mConfig;
  final PagedStorage<T> mStorage;
}

// ItemKeyedDataSource.LoadInitialParams.java
public static class LoadInitialParams<Key> {
  public final Key requestedInitialKey;
  public final int requestedLoadSize;
  public final boolean placeholdersEnabled;
}
```

> 上文说到 **几乎没有用到单例模式**，实际上线程切换的设计有些许例外，但其本身依然可以通过`Builder`进行依赖注入以覆盖默认的线程获取逻辑。

通过 **依赖注入** 保证了对象的实例所需依赖有迹可循，类与类之间的依赖关系非常清晰，而实例化的对象内部 **成员的不可变** 也极大保证了`PagedList`分页数据的线程安全。

### 2、构建懒加载的LiveData

对于被观察者而言，只有当真正被订阅的时候，其数据的更新才有意义。换句话说，当开发者构建出一个`LiveData<PagedList>`时候，这时立即通过后台线程开始异步请求分页数据是没有意义的。

> 反过来理解，若没有订阅就请求数据，当真正订阅的时候，`DataSource`中的数据已经过时了，这时还需要重新请求拉取最新数据，这样之前的一系列行为就没有意义了。

真正的请求应该放在`LiveData.observe()`的时候，即被订阅时才去执行，笔者这里更偏向于称其为“懒加载”——如果读者对`RxJava`比较熟悉的话，会发现这和`Observable.defer()`操作符概念比较相似：

![](https://raw.githubusercontent.com/qingmei2/qingmei2-blogs-art/master/android/jetpack/paging/thinking_in_android/image.n4fl2yvwo9.png)

那么，如何构建“懒加载”的`LiveData<PagedList>`呢?`Google`的设计者使用了`ComputableLiveData`类对`LiveData`的数据发射行为进行了包装：

```
// @hide
public abstract class ComputableLiveData<T> {}
```

这是一个隐藏的类，开发者一般不能直接使用它，但它被应用的地方可不少，`Room`组件生成的源码中也经常可以看到它的身影。

用一句话描述`ComputableLiveData`的定义，笔者觉得 **LiveData的数据源** 比较适合，感兴趣的读者可以仔细研究一下它的源码，笔者有机会会为它单独开一篇文章，这里不继续展开。

总之，通过`ComputableLiveData`类，`Paging`实现了订阅时才执行异步任务的功能，更大程度上减少了做无用功的情况。

### 3、为分页数据赋予生命周期

### 4、提供更多响应式类型的支持
