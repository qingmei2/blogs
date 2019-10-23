## 三、工作流程原理概述

`Paging`幕后是如何工作的？

接下来，笔者将针对`Paging`分页组件的工作流程进行系统性的描述，探讨`Paging`是 **如何实现异步分页数据的加载和响应** 的。

为了便于理解，笔者将整个流程拆分为三个步骤，并为每个步骤绘制对应的一张流程图，这三个步骤分别是：

* 1.初次创建流程
* 2.UI渲染和分页加载流程
* 3.刷新数据源流程

### 1.初次创建流程

![](https://raw.githubusercontent.com/qingmei2/qingmei2-blogs-art/master/android/jetpack/paging/thinking_in_android/image.r1xngwofsl.png)

如图所示，我们定义了`ViewModel`和`Repository`，`Repository`内部实现了`App`的数据加载的逻辑，而其左侧的`ViewModel`则负责与`UI`组件的通信。

`Repository`负责为`ViewModel`中的`LiveData<PagedList<User>>`进行创建，因此，开发者需要创建对应的`PagedList.Config`分页配置对象和`DataSource.Factory`数据源的工厂，并通过调用`LivePagedListBuilder`相关的`API`创建出一个`LiveData<PagedList<User>>`。

当`LiveData`一旦被订阅，`Paging`将会尝试创建一个`PagedList`，同时，数据源的工厂`DataSource.Factory`也会创建一个`DataSource`，并交给`PagedList`持有该`DataSource`。

这时候`PagedList`已经被成功的创建了，但是此时的`PagedList`内部只持有了一个`DataSource`，却并没有持有任何数据，这意味着观察者角色的`UI`层即将接收到一个空数据的`PagedList`。

这没有任何意义，因此我们更希望`PagedList`第一次传递到`UI`层级的同时，已经持有了初始的列表数据（即`InitialLoadSizeHint`）；因此，`Paging`尝试在后台线程中通过`DataSource`对`PagedList`内部的数据列表进行初始化。

现在，`PagedList`第一次创建完毕，并持有属于自己的`DataSource`和初始的列表数据，通过`LiveData`这个管道，即将向`UI`层迈出属于自己的第一个脚印。

### 2.UI渲染和分页加载流程

![](https://raw.githubusercontent.com/qingmei2/qingmei2-blogs-art/master/android/jetpack/paging/thinking_in_android/image.1bcu7jz8e0d.png)

通过内部线程的切换，`PagedList`从后台线程切换到了`UI`线程，通过`LiveData`抵达了`UI`层级，也就是我们通常说的`Activity`或者`Fragment`中。

读者应该有印象，在上文的示例代码中，`Activity`观察到`PagedList`后，会通过`PagedListAdapter.submitList()`函数将`PagedList`进行注入。`PagedListAdapter`第一次接收到`PagedList`后，就会对`UI`进行渲染。

当用户尝试对屏幕中的列表进行滚动时，我们接收到了需要加载更多数据的信号，这时，`PagedList`在内部主动触发数据的加载，数据源提供了更多的数据，`PagedList`接收到之后将会主动触发`RecyclerView`的更新，用户通过`RecyclerView`原生动画观察到了更多的列表`Item`。

### 3.刷新数据源流程

![](https://raw.githubusercontent.com/qingmei2/qingmei2-blogs-art/master/android/jetpack/paging/thinking_in_android/progress03.y5fcy8shi39.jpg)

当数据发生了更新，`Paging`幕后又做了哪些工作呢？

正如前文所说，**数据是动态的**， 假设用户通过操作添加了一个联系人，这时数据库中的数据集发生了更新。

因此，这时屏幕中`RecyclerView`对应的`PagedList`和`DataSource`已经没有失效了，因为`DataSource`中的数据是之前数据库中数据的快照，数据库内部进行了更新，`PagedList`从旧的`DataSource`中再取数据毫无意义。

因此，`Paging`组件接收到了数据失效的信号，这意味着生产者需要重新构建一个`PagedList`，因此`DataSource.Factory`再次提供新版本的数据源`DataSource V2`——其内部持有了最新数据的快照。

在创建新的`PagedList`的时候，针对`PagedList`内部的初始化需要慎重考虑，因为初始化的数据需要根据用户当前屏幕中所在的位置（`position`）进行加载。

通过`LiveData`，`UI`层级再次观察到了新的`PagedList`，并再次通过`submitList()`函数注入到`PagedListAdapter`中。

和初次的数据渲染不同，这一次我们使用到了`PagedListAdapter`内部的`AsyncPagedListDiffer`对两个数据集进行差异性计算——这避免了`notifyDataSetChanged()`的滥用，同时，差异性计算的任务被切换到了后台线程中执行，一旦计算出差异性结果，新的`PagedList`会替换旧的`PagedList`，并对列表进行 **增量更新**。

## 四、DataSource数据源简介

`Paging`分页组件的设计中，`DataSource`是一个非常重要的模块。顾名思义，`DataSource<Key, Value>`中的`Key`对应数据加载的条件，`Value`对应数据集的实际类型， 针对不同场景，`Paging`的设计者提供了三种不同类型的`DataSource`抽象类:

* `PositionalDataSource<T>`
* `ItemKeyedDataSource<Key, Value>`
* `PageKeyedDataSource<Key, Value>`

接下来我们分别对其进行简单的介绍。

> 本章节涉及的知识点非常重要，但不作为本文的重点，笔者将在该系列的下一篇文章中针对`DataSource`的设计与实现进行更细节的探究，欢迎关注。

### 1.PositionalDataSource

`PositionalDataSource<T>`是最简单的`DataSource`类型，顾名思义，其通过数据所处当前数据集快照的位置（`position`）提供数据。

`PositionalDataSource<T>`适用于 **目标数据总数固定**，通过特定的位置加载数据，这里`Key`是`Integer`类型的位置信息，并且被内置固定在了`PositionalDataSource<T>`类中，`T`即数据的类型。

最容易理解的例子就是本文的联系人列表，其所有的数据都来自本地的数据库，这意味着，数据的总数是固定的，我们总是可以根据当前条目的`position`映射到`DataSource`中对应的一个数据。

`PositionalDataSource<T>`也正是`Room`幕后实现的功能，使用`Room`为什么可以避免`DataSource`的配置，通过`dao`中的接口就能返回一个`DataSource.Factory`？

来看`Room`组件配置的`dao`对应编译期生成的源码：

```Java
// 1.Room自动生成了 DataSource.Factory
@Override
 public DataSource.Factory<Integer, Student> getAllStudent() {
   // 2.工厂函数提供了PositionalDataSource
   return new DataSource.Factory<Integer, Student>() {
     @Override
     public PositionalDataSource<Student> create() {
       return new PositionalDataSource<Student>(__db, _statement, false , "Student") {
         // ...
       };
     }
   };
 }
```

### 2.ItemKeyedDataSource

`ItemKeyedDataSource<Key, Value>`适用于目标数据的加载依赖特定条目的信息，比如需要根据第N项的信息加载第N+1项的数据，传参中需要传入第N项的某些信息时。

同样拿联系人列表举例，另外的一种分页加载方式是通过上一个联系人的`name`作为`Key`请求新一页的数据，因为联系人`name`字母排序的原因，`DataSource`很容易针对一个`name`检索并提供接下来新一页的联系人数据——比如根据`Alice`找到下一个用户`Bob`（`A -> B`）。

### 3.PageKeyedDataSource

更多的网络请求`API`中，服务器返回的数据中都会包含一个`String`类型类似`nextPage`的字段，以表示当前页数据的下一页数据的接口（比如`Github`的`API`），这种分页数据加载的方式正是`PageKeyedDataSource<Key, Value>`的拿手好戏。

这是日常开发中用到最多的`DataSource`类型，和`ItemKeyedDataSource<Key, Value>`不同的是，前者的数据检索关系是单个数据与单个数据之间的，后者则是每一页数据和每一页数据之间的。

同样拿联系人列表举例，这种分页加载方式是按照页码进行数据加载的，比如一次请求15条数据，服务器返回数据列表的同时会返回下一页数据的`url`（或者页码），借助该参数请求下一页数据成功后，服务器又回返回下下一页的`url`，以此类推。

总的来说，`DataSource`针对不同种数据分页的加载策略提供了不同种的抽象类以方便开发者调用，很多情况下，同样的业务使用不同的`DataSource`都能够实现，开发者按需取用即可。

## 五、最佳实践

现在读者对多种不同的数据源`DataSource`有了简单的了解，先抛开 **分页列表** 的业务不谈，我们思考另外一个问题：

> 当列表的数据通过多个层级 **网络请求**（`Network`） 和 **本地缓存** （`Database`）进行加载该怎么处理？

回答这个问题，需要先思考另外一个问题：

> `Network`+`Database`的解决方案有哪些优势？

### 1.优势

读者认真思考可得，`Network`+`Database`的解决方案优点如下：

* 1.非常优秀的离线模式支持，即使用户设备并没有链接网络，本地缓存依然可以带来非常不错的使用体验；
* 2.数据的快速恢复，如果异常导致`App`的终止，本地缓存可以对页面数据进行快速恢复，大幅减少流量的损失，以及加载的时间。
* 3.两者的配合的效果总是相得益彰。

看起来`Network`+`Database`是一个非常不错的数据加载方案，那么为什么大多数场景并没有使用本地缓存呢？

主要原因是开发成本——本地缓存的搭建总是需要额外的代码，不仅如此，更重要的原因是，**数据交互的复杂性也会导致额外的开发成本**。

### 2.复杂的交互模型

为什么说`Network`+`Database`会导致 **数据交互的复杂性** ？

让我们回到本文的 **联系人列表** 的示例中，这个示例中，所有联系人数据都来自 **本地缓存**，因此读者可以很轻易的构建出该功能的整体结构：

![](https://raw.githubusercontent.com/qingmei2/qingmei2-blogs-art/master/android/jetpack/paging/thinking_in_android/image.8dz585ajgcb.png)

如图所示，`ViewModel`中的数据总是由`Database`提供，如果把数据源从`Database`换成`Network`，数据交互的模型也并没有什么区别—— **数据源总是单一的**。

那么，当数据的来源不唯一时——即`Network`+`Database`的数据加载方案中会有哪些问题呢？

我们来看看常规的实现方案的数据模型：

![](https://raw.githubusercontent.com/qingmei2/qingmei2-blogs-art/master/android/jetpack/paging/thinking_in_android/image.4i88e2kodql.png)

如图所示，`ViewModel`尝试加载数据时，总是会先进行网络判断，若网络未连接，则展示本地缓存，否则请求网络，并且在网络请求成功时，将数据保存本地。

乍得一看，这种方案似乎并没有什么问题，实际上却有两个非常大的弊端：

#### 2.1 业务并非这么简单

首先，通过一个`boolean`类型的值就能代表网络连接的状态吗？显而易见，答案是否定的。

实际上，在某些业务场景下，服务器的连接状态可以是更为复杂的，比如接收到了部分的数据包？比如某些情况下网络请求错误，这时候是否需要重新展示本地缓存？

若涉及到网络请求的重试则更复杂，成功展示网络数据，再次失败展示缓存——业务越来越复杂，我们甚至会逐渐沉浸其中无法自拔，最终醒悟，**这种数据的交互模型完全不够用了** 。

#### 2.2 无用的本地缓存

另外一个很明显的弊端则是，当网络连接状态良好的时候，用户看到的数据总是服务器返回的数据。

这种情况下，请求的数据再次存入本地缓存似乎毫无意义，因为网络环境的通畅，`Database`中的缓存从来未作为数据源被展示过。

### 3.使用单一数据源

使用 **单一数据源** (`single source of truth`)的好处不言而喻，正如上文所阐述的，**多个数据源** 反而会将业务逻辑变得越来越复杂，因此，我们设计出这样的模型：

![](https://raw.githubusercontent.com/qingmei2/qingmei2-blogs-art/master/android/jetpack/paging/thinking_in_android/image.tilqwkzdjk.png)

`ViewModel`如果响应`Database`中的数据变更，且`Database`作为唯一的数据来源？

其思路是：`ViewModel`只从`Database`中取得数据，当`Database`中数据不够时，则向`Server`请求网络数据，请求成功，数据存入`Database`，`ViewModel`观察到`Database`中数据的变更，并更新到`UI`中。

这似乎无法满足上文中的需求？读者认真思考可知，其实是没问题的，当网络连接发生故障时，这时向服务端请求数据失败，并不会更新`Database`，因此`UI`展示的正是期望的本地缓存。

`ViewModel`仅仅响应`Database`中数据的变更，这种使用 **单一数据源** 的方式让复杂的业务逻辑简化了很多。

### 4.分页列表的最佳实践

现在我们理解了 **单一数据源** 的好处，该方案在分页组件中也同样适用，我们唯一需要实现的是，如何主动触发服务端数据的请求？

这是当然的，因为`Server`总是需要一个触发网络请求的信号，否则列表所展示的永远是`Database`中的数据——别忘了，`ViewModel`和`Server`之间并没有任何关系。

简单
