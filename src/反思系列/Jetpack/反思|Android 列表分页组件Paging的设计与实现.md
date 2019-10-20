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

### 1.1 服务端组件、数据库、内存

为什么说 **服务端组件**、**数据库** 以及 **内存** 是非常重要的三个层级呢？

首先，开发者为当前页面创建了一个`ViewModel`，并通过成员变量在 **内存** 中持有了一组联系人数据，因为`ViewModel`组件的原因，即使页面配置发生了改变（比如屏幕的旋转），数据依然会被保留下来。

而 **数据库** 的作用则保证了`App`即使在离线环境下，用户依然可以看到一定的内容——显然对于上图中的页面（联系人列表）而言，本地缓存是非常有意义的。

对于绝大多数列表而言，**服务端** 往往意味着是数据源，每当用户执行刷新操作，`App`都应当尝试向服务端请求最新的数据，并将最新的数据存入 **数据库**，并随之展示在`UI`上。

通常情况下，这三个层级并非同时都是必要的，读者需正确理解三者各自不同的使用场景。

现在，借助于 **服务端组件**、**数据库** 以及 **内存**，开发者将数据展示在`RecyclerView`上，这似乎已经是正解了。

### 1.2 问题在哪？

到目前为止，问题还没有完全暴露出来。

我们忽视了一个非常现实的问题，那就是 **数据是动态的** ——这意味着，每当数据发生了更新（比如用户进行了下拉刷新操作），开发者都需要将最新的数据响应在`UI`上。

这意味着，当某个用户的联系人列表中有10000个条目时，每次数据的更新，都会对所有的数据进行重建——从而导致 **性能非常低下**，用户看到的只是屏幕中的几条联系人信息，为此要重新创建10000个条目？用户显然无法接受。

因此，分页组件的设计势在必行。

### 1.3 整理需求

#### 1、简单易用

上文我们谈到，**UI响应数据的变更**，这种情况下，使用 **观察者模式** 是一个不错的主意，比如`LiveData`、`RxJava`甚至自定义一个接口等等，开发者仅需要观察每次数据库中数据的变更，并进行`UI`的更新：

```Kotlin
class MyViewModel : ViewModel() {
  val users: LiveData<List<User>>
}
```

新的组件我们也希望能拥有同样的便利，比如使用`LiveData`或者`RxJava`，并进行订阅处理数据的更新—— **简单** 且 **易用**。

#### 2、处理更多层级

我们希望新的组件能够处理多层，我们希望列表展示 **服务器** 返回的数据、 或者 **数据库** 中的数据，并将其放入UI中。

#### 3、性能

新的组件必须保证足够的快，不做任何没必要的行为，为了保证效率，繁重的操作不要直接放在`UI`线程中处理。

#### 4、感知生命周期

如果可能，新的组件需要能够对生命周期进行感知，就像`LiveData`一样，如果页面并不在屏幕的可视范围内，组件不应该工作。

#### 5、足够灵活

足够的灵活性非常重要——每个项目都有不同的业务，这意味着不同的`API`、不同的数据结构，新的组件必须保证能够应对所有的业务场景。

> 这一点并非必须，但是对于设计者来说难度不小，这意味着需要将不同的业务中的共同点抽象出来，并保证这些设计适用在任何场景中。

定义好了需求，在正式开始设计Paging之前，首先我们先来回顾一下，普通的列表如何实现数据的动态更新的。

### 1.4 普通列表的实现方式

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

乍得一看似乎理所当然，但实际需求中有一个问题，这里的定义是模糊不清的——对于分页数据而言，不同的业务场景，所需要的分页策略是不同的。那么什么是分页策略呢？

最直接的一点是每页数据的加载数量`PageSize`，不同的项目都会自行规定每页数据量的大小，一页请求15个数据还是20个数据？显然我们目前的代码无法进行配置，这是不合理的。

### 2.数据源: DataSource

我们需要一个角色为`PagedList`容器提供分页数据，那就是数据源`DataSource`。

那么什么是`DataSource`呢？它不应该是 **数据库数据** 或者 **服务端数据**， 而应该是 **数据库数据** 或者 **服务端数据** 的一个快照（`Snapshot`）。

因此，每当`Paging`被告知需要更多数据：“Hi，我需要第45-60个的数据！”——数据源`DataSource`就会将当前`Snapshot`对应索引的数据交给`PagedList`。

但是，每当我们需要构建一个新的`PagedList`时——比如数据已经失效，`DataSource`中旧的数据没有意义了，因此`DataSource`也需要被重置。

在代码中，这意味着新的`DataSource`对象被创建，因此，我们需要提供的不是`DataSource`，而是提供`DataSource`的工厂。

> 为什么要提供`DataSource.Factory`而不是一个`DataSource`? 复用这个`DataSource`不可以吗，当然可以，但是将`DataSource`设置为`immutable`(不可变)会避免更多的未知因素。

![](https://raw.githubusercontent.com/qingmei2/qingmei2-blogs-art/master/android/jetpack/paging/thinking_in_android/image.5ay4a09k0y5.png)
