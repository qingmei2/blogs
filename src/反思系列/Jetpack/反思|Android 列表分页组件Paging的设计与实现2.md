### 三、工作流程原理概述

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

和初次的数据渲染不同，这一次我们使用到了`PagedListAdapter`内部的`AsyncPagedListDiffer`对两个数据集进行差异性计算——这避免了`notifyDataSetChanged()`的滥用，同时，差异性计算的任务被切换到了后台线程中执行，一旦计算出差异性结果，新的`PagedList`会替换旧的`PagedList`，并对列表进行局部的增量更新。
