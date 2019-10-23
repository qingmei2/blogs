# 反思|Android 列表分页组件Paging的设计与实现：架构描述

## 前言

本文将对`Paging`分页组件的设计和实现进行一个系统整体的概述，**强烈建议** 读者将本文作为学习`Paging` **阅读优先级最高的文章**，所有其它的`Paging`中文博客阅读优先级都应该靠后。

本文篇幅 **极长**，整体结构思维导图如下：

![](https://raw.githubusercontent.com/qingmei2/qingmei2-blogs-art/master/android/jetpack/paging/image.y4rdl3xz6rj.png)

## 一、起源

手机应用中，列表是常见的界面构成元素，而对于Android开发者而言，`RecyclerView`是实现列表的不二选择。

在正式讨论`Paging`和列表分页功能之前，我们首先看看对于一个普通的列表，开发者如何通过代码对其进行建模：

![](https://raw.githubusercontent.com/qingmei2/qingmei2-blogs-art/master/android/jetpack/paging/thinking_in_android/image.hrp8e3vf6lk.png)

如图所示，针对这样一个简单 **联系人界面** 的建模，我们引出3个重要的层级：

### 1.服务端组件、数据库、内存

为什么说 **服务端组件**、**数据库** 以及 **内存** 是非常重要的三个层级呢？

首先，开发者为当前页面创建了一个`ViewModel`，并通过成员变量在 **内存** 中持有了一组联系人数据，因为`ViewModel`组件的原因，即使页面配置发生了改变（比如屏幕的旋转），数据依然会被保留下来。

而 **数据库** 的作用则保证了`App`即使在离线环境下，用户依然可以看到一定的内容——显然对于上图中的页面（联系人列表）而言，本地缓存是非常有意义的。

对于绝大多数列表而言，**服务端** 往往意味着是数据源，每当用户执行刷新操作，`App`都应当尝试向服务端请求最新的数据，并将最新的数据存入 **数据库**，并随之展示在`UI`上。

通常情况下，这三个层级并非同时都是必要的，读者需正确理解三者各自不同的使用场景。

现在，借助于 **服务端组件**、**数据库** 以及 **内存**，开发者将数据展示在`RecyclerView`上，这似乎已经是正解了。

### 2.问题在哪？

到目前为止，问题还没有完全暴露出来。

我们忽视了一个非常现实的问题，那就是 **数据是动态的** ——这意味着，每当数据发生了更新（比如用户进行了下拉刷新操作），开发者都需要将最新的数据响应在`UI`上。

这意味着，当某个用户的联系人列表中有10000个条目时，每次数据的更新，都会对所有的数据进行重建——从而导致 **性能非常低下**，用户看到的只是屏幕中的几条联系人信息，为此要重新创建10000个条目？用户显然无法接受。

因此，分页组件的设计势在必行。

### 3.整理需求

#### 3.1、简单易用

上文我们谈到，**UI响应数据的变更**，这种情况下，使用 **观察者模式** 是一个不错的主意，比如`LiveData`、`RxJava`甚至自定义一个接口等等，开发者仅需要观察每次数据库中数据的变更，并进行`UI`的更新：

```Kotlin
class MyViewModel : ViewModel() {
  val users: LiveData<List<User>>
}
```

新的组件我们也希望能拥有同样的便利，比如使用`LiveData`或者`RxJava`，并进行订阅处理数据的更新—— **简单** 且 **易用**。

#### 3.2、处理更多层级

我们希望新的组件能够处理多层，我们希望列表展示 **服务器** 返回的数据、 或者 **数据库** 中的数据，并将其放入UI中。

#### 3.3、性能

新的组件必须保证足够的快，不做任何没必要的行为，为了保证效率，繁重的操作不要直接放在`UI`线程中处理。

#### 3.4、感知生命周期

如果可能，新的组件需要能够对生命周期进行感知，就像`LiveData`一样，如果页面并不在屏幕的可视范围内，组件不应该工作。

#### 3.5、足够灵活

足够的灵活性非常重要——每个项目都有不同的业务，这意味着不同的`API`、不同的数据结构，新的组件必须保证能够应对所有的业务场景。

> 这一点并非必须，但是对于设计者来说难度不小，这意味着需要将不同的业务中的共同点抽象出来，并保证这些设计适用在任何场景中。

定义好了需求，在正式开始设计Paging之前，首先我们先来回顾一下，普通的列表如何实现数据的动态更新的。

### 4.普通列表的实现方式

我们依然通过 **联系人列表** 作为示例，来描述普通列表 **如何响应数据的动态更新**。

首先，我们需要定义一个`Dao`，这里我们使用了`Room`组件用于 **数据库** 中联系人的查询：

```Kotlin
@Dao
interface UserDao {
  @Query("SELECT * FROM user")
  fun queryUsers(): LiveData<List<User>>
}
```

这里我们返回的是一个`LiveData`，正如我们前文所言，构建一个可观察的对象显然会让数据的处理更加容易。

接下来我们定义好`ViewModel`和`Activity`:

```Kotlin
class MyViewModel(val dao: UserDao) : ViewModel() {
  // 1.定义好可观察的LiveData
  val users: LiveData<List<User>> = dao.queryUsers()
}

class MyActivity : Activity {
  val myViewModel: MyViewModel
  val adapter: ListAdapter

  fun onCreate(bundle: Bundle?) {
    // 2.在Activity中对LiveData进行订阅
    myViewModel.users.observe(this) {
      // 3.每当数据更新，计算新旧数据集的差异，对列表进行更新
      adapter.submitList(it)
    }
  }    
}
```

 这里我们使用到了`ListAdapter`，它是官方基于`RecyclerView.Adapter`的`AsyncListDiffer`封装类，其内创建了`AsyncListDiffer`的示例，以便在后台线程中使用`DiffUtil`计算新旧数据集的差异，从而节省`Item`更新的性能。

> 本文默认读者对`ListAdapter`一定了解，如果不是很熟悉，请参考`DiffUtil`、`AsyncListDiffer`、`ListAdapter`等相关知识点的文章。

此外，我们还需要在`ListAdapter`中声明`DiffUtil.ItemCallback`，对数据集的差异计算的逻辑进行补充：

```Kotlin
class MyAdapter(): ListAdapter<User, UserViewHolder>(
  object: DiffUtil.ItemCallback<User>() {
    override fun areItemsTheSame(oldItem: User, newItem: User)
        = oldItem.id == newItem.id
    override fun areContentsTheSame(oldItem: User, newItem: User)
        = oldItem == newItem   
  }
) {
  // ...
}
```

That's all, 接下来我们开始思考，新的分页组件应该是什么样的。

## 二、分页组件简介

### 1.核心类：PagedList

上文提到，一个普通的`RecyclerView`展示的是一个列表的数据，比如`List<User>`，但在列表分页的需求中，`List<User>`明显就不太够用了。

为此，`Google`设计出了一个新的角色`PagedList`，顾名思义，该角色的意义就是 **分页列表数据的容器 ** 。

> 既然有了`List`，为什么需要额外设计这样一个`PagedList`的数据结构？本质原因在于加载分页数据的操作是异步的 ，因此定义`PagedList`的第二个作用是 **对分页数据的异步加载** ,这个我们后文再提。

现在，我们的`ViewModel`现在可以定义成这样，因为`PagedList`也作为列表数据的容器（就像`List<User>`一样）：

```Kotlin
class MyViewModel : ViewModel() {
  // before
  // val users: LiveData<List<User>> = dao.queryUsers()

  // after
  val users: LiveData<PagedList<User>> = dao.queryUsers()
}
```

在`ViewModel`中，开发者可以轻易通过对`users`进行订阅以响应分页数据的更新，这个`LiveData`的可观察者是通过`Room`组件创建的，我们来看一下我们的`dao`:

```Kotlin
@Dao
interface UserDao {
  // 注意，这里 LiveData<List<User>> 改成了 LiveData<PagedList<User>>  
  @Query("SELECT * FROM user")
  fun queryUsers(): LiveData<PagedList<User>>  
}
```

乍得一看似乎理所当然，但实际需求中有一个问题，这里的定义是模糊不清的——对于分页数据而言，不同的业务场景，所需要的相关配置是不同的。那么什么是分页相关配置呢？

最直接的一点是每页数据的加载数量`PageSize`，不同的项目都会自行规定每页数据量的大小，一页请求15个数据还是20个数据？显然我们目前的代码无法进行配置，这是不合理的。

### 2.数据源: DataSource及其工厂

回答这个问题之前，我们还需要定义一个角色，用来为`PagedList`容器提供分页数据，那就是数据源`DataSource`。

什么是`DataSource`呢？它不应该是 **数据库数据** 或者 **服务端数据**， 而应该是 **数据库数据** 或者 **服务端数据** 的一个快照（`Snapshot`）。

每当`Paging`被告知需要更多数据：“Hi，我需要第45-60个的数据！”——数据源`DataSource`就会将当前`Snapshot`对应索引的数据交给`PagedList`。

但是我们需要构建一个新的`PagedList`的时候——比如数据已经失效，`DataSource`中旧的数据没有意义了，因此`DataSource`也需要被重置。

在代码中，这意味着新的`DataSource`对象被创建，因此，我们需要提供的不是`DataSource`，而是提供`DataSource`的工厂。

> 为什么要提供`DataSource.Factory`而不是一个`DataSource`? 复用这个`DataSource`不可以吗，当然可以，但是将`DataSource`设置为`immutable`(不可变)会避免更多的未知因素。

重新整理思路，我们如何定义`Dao`中接口的返回值呢？

```Kotlin
@Dao
interface UserDao {
  // Int 代表按照数据的位置（position）获取数据
  // User 代表数据的类型
  @Query("SELECT * FROM user")
  fun queryUsers(): DataSource.Factory<Int, User>
}
```

返回的是一个数据源的提供者`DataSource.Factory`，页面初始化时，会通过工厂方法创建一个新的`DataSource`，这之后对应会创建一个新的`PagedList`，每当`PagedList`想要获取下一页的数据，数据源都会根据请求索引进行数据的提供。

当数据失效时，`DataSource.Factory`会再次创建一个新的`DataSource`，其内部包含了最新的数据快照（本案例中代表着数据库中的最新数据），随后创建一个新的`PagedList`，并从`DataSource`中取最新的数据进行展示——当然，这之后的分页流程都是相同的，无需再次复述。

笔者绘制了一幅图用于描述三者之间的关系，读者可参考上述文字和图片加以理解：

![](https://raw.githubusercontent.com/qingmei2/qingmei2-blogs-art/master/android/jetpack/paging/thinking_in_android/image.5ay4a09k0y5.png)

### 3.串联两者：PagedListBuilder

回归第一小节的那个问题，分页相关业务如何进行配置？我们虽然介绍了为`PagedList`提供数据的`DataSource`，但这个问题似乎还是没有得到解决。

此外，现在`Dao`中接口的返回值已经是`DataSource.Factory`，而`ViewModel`中的成员被观察者则是`LiveData<PagedList<User>>`类型，如何 **将数据源的工厂和`LiveData<PagedList>`进行串联**  ？

因此我们还需要定义一个新的角色`PagedListBuilder`，开发者将 **数据源工厂** 和 **相关配置** 统一交给`PagedListBuilder`，即可生成对应的`LiveData<PagedList<User>>`:

```Kotlin
class MyViewModel(val dao: UserDao) : ViewModel() {
  val users: LiveData<PagedList<User>>

  init {
    // 1.创建DataSource.Factory
    val factory: DataSource.Factory = dao.queryUsers()

    // 2.通过LivePagedListBuilder配置工厂和pageSize, 对users进行实例化
    users = LivePagedListBuilder(factory, 30).build()
  }
}
```

如代码所示，我们在`ViewModel`中先通过`dao`获取了`DataSource.Factory`，工厂创建数据源`DataSource`，后者为`PagedList`提供列表所需要的数据；此外，另外一个`Int`类型的参数则制定了每页数据加载的数量，这里我们指定每页数据数量为30。

我们成功创建了一个`LiveData<PagedList<User>>`的可观察者对象，接下来的步骤读者驾轻就熟，只不过我们这里使用的是`PagedListAdapter`：

```Kotlin
class MyActivity : Activity {
  val myViewModel: MyViewModel
  // 1.这里我们使用PagedListAdapter
  val adapter: PagedListAdapter

  fun onCreate(bundle: Bundle?) {
    // 2.在Activity中对LiveData进行订阅
    myViewModel.users.observe(this) {
      // 3.每当数据更新，计算新旧数据集的差异，对列表进行更新
      adapter.submitList(it)
    }
  }    
}
```

`PagedListAdapter`内部的实现和普通列表`ListAdapter`的代码几乎完全相同：

```Kotlin
// 几乎完全相同的代码，只有继承的父类不同
class MyAdapter(): PagedListAdapter<User, UserViewHolder>(
  object: DiffUtil.ItemCallback<User>() {
    override fun areItemsTheSame(oldItem: User, newItem: User)
        = oldItem.id == newItem.id
    override fun areContentsTheSame(oldItem: User, newItem: User)
        = oldItem == newItem   
  }
) {
  // ...
}
```

> 准确的来说，两者内部的实现还有微弱的区别，前者`ListAdapter`的`getItem()`函数的返回值是`User`,而后者`PagedListAdapter`返回值应该是`User?`(Nullable),其原因我们会在下面的`Placeholder`部分进行描述。

### 4.更多可选配置：PagedList.Config

目前的介绍中，分页的功能似乎已经实现完毕，但这些在现实开发中往往不够，产品业务还有更多细节性的需求。

在上一小节中，我们通过`LivePagedListBuilder`对`LiveData<PagedList<User>>`进行创建，这其中第二个参数是 **分页组件的配置**，代表了每页加载的数量（`PageSize`） ：

```Kotlin
// before
val users: LiveData<PagedList<User>> = LivePagedListBuilder(factory, 30).build()
```

读者应该理解，**分页组件的配置** 本身就是抽象的，`PageSize`并不能完全代表它，因此，设计者额外定义了更复杂的数据结构`PagedList.Config`，以描述更细节化的配置参数：

```Kotlin
// after
val config = PagedList.Config.Builder()
      .setPageSize(15)              // 分页加载的数量
      .setInitialLoadSizeHint(30)   // 初次加载的数量
      .setPrefetchDistance(10)      // 预取数据的距离
      .setEnablePlaceholders(false) // 是否启用占位符
      .build()

// API发生了改变
val users: LiveData<PagedList<User>> = LivePagedListBuilder(factory, config).build()
```

对复杂业务配置的`API`设计来说，**建造者模式** 显然是不错的选择。

接下来我们简单了解一下，这些可选的配置分别代表了什么。

#### 4.1.分页数量：PageSize

最易理解的配置，分页请求数据时，开发者总是需要定义每页加载数据的数量。

#### 4.2.初始加载数量：InitialLoadSizeHint

定义首次加载时要加载的`Item`数量。

此值通常大于`PageSize`，因此在初始化列表时，该配置可以使得加载的数据保证屏幕可以小范围的滚动。

如果未设置，则默认为`PageSize`的三倍。

#### 4.3.预取距离：PrefetchDistance

顾名思义，该参数配置定义了列表当距离加载边缘多远时进行分页的请求，默认大小为`PageSize`——即距离底部还有一页数据时，开启下一页的数据加载。

若该参数配置为0，则表示除非明确要求，否则不会加载任何数据，通常不建议这样做，因为这将导致用户在滚动屏幕时看到占位符或列表的末尾。

#### 4.4.是否启用占位符：PlaceholderEnabled

该配置项需要传入一个`boolean`值以决定列表是否开启`placeholder`（占位符），那么什么是`placeholder`呢？

我们先来看未开启占位符的情况：

![](https://raw.githubusercontent.com/qingmei2/qingmei2-blogs-art/master/android/jetpack/paging/thinking_in_android/placeholder01.zi87tn0olm.gif)

如图所示，没有开启占位符的情况下，列表展示的是当前所有的数据，请读者重点观察图片右侧的滚动条，当滚动到列表底部，成功加载下一页数据后，滚动条会从长变短，这意味着，新的条目成功实装到了列表中。一言以蔽之，**未开启占位符的列表，条目的数量和`PagedList`中数据数量是一致的。**

接下来我们看一下开启了占位符的情况：

![](https://raw.githubusercontent.com/qingmei2/qingmei2-blogs-art/master/android/jetpack/paging/thinking_in_android/placeholder02.7exa4kjff3w.gif)

如图所示，**开启了占位符的列表，条目的数量和`DataSource`中数据的总量是一致的。** 这并不意味着列表从`DataSource`一次加载了大量的数据并进行渲染，所有业务依然交给`Paging`进行分页处理。

当用户滑动到了底部尚未加载的数据时，开发者会看到还未渲染的条目，这是理所当然的，`PagedList`的分页数据加载是异步的，这时对于`Item`的来说，要渲染的数据为`null`，因此开发者需要配置占位符，当数据未加载完毕时，UI如何进行渲染——这也正是为何上文说到，对于`PagedListAdapter`来说，`getItem()`函数的返回值是可空的`User?`，而不是`User`。

随着`PagedList`下一页数据的异步加载完毕，伴随着`RecyclerView`的原生动画，新的数据会被重新覆盖渲染到`placeholder`对应的条目上，就像`gif`图展示的一样。

#### 4.5.关于Placeholder

这里我专门开一个小节谈谈关于`placeholder`，因为这个机制和我们传统的分页业务似乎有所不同，但`Google`的工程师们认为在某些业务场景下，该配置确实很有用。

开启了占位符，用户总是可以快速的滑动列表，因为列表“持有”了整个数据集，因此不会像未开启占位符时，滑动到底部而被迫暂停滚动，直到新的数据的加载完毕才能继续浏览。**顺畅的操作总比期望之外的阻碍要好得多** 。

此外，开启了占位符意味着用户与 **加载指示器** 彻底告别，类似一个 **正在加载更多...** 的提示标语或者一个简陋的`ProgressBar`效果真的会提升用户体验吗？也许答案是否定的，相比之下，用户应该更喜欢一个灰色的占位符，并等待它被新的数据渲染。

但缺点也随之而来，首先，占位符的条目高度应该和正确的条目高度一致，在某些需求中，这也许并不符合，这将导致渐进性的动画效果并不会那么好。

其次，对于开发者而言，开启占位符意味着需要对`ViewHolder`进行额外的代码处理，数据为`null`或者不为`null`？两种情况下的条目渲染逻辑都需要被添加。

最后，这是一个限制性的条件，您的`DataSource`数据源内部的数据数量必须是确定的，比如通过`Room`从本地获取联系人列表；而当数据通过网络请求获取的话，这时数据的数量是不确定的，不开启`Placeholder`反而更好。

### 5.更多观察者类型的配置

在本文的示例中，我们建立了一个`LiveData<PagedList<User>>`的可观察者对象供用户响应数据的更新，实际上组件的设计应该面向提供对更多优秀异步库的支持，比如`RxJava`。

因此，和`LivePagedListBuilder`一样，设计者还提供了`RxPagedListBuilder`，通过`DataSource`数据源和`PagedList.Config`以构建一个对应的`Observable`:

```Kotlin
// LiveData support
val users: LiveData<PagedList<User>> = LivePagedListBuilder(factory, config).build()

// RxJava support
val users: Observable<PagedList<User>> = RxPagedListBuilder(factory, config).buildObservable()
```
