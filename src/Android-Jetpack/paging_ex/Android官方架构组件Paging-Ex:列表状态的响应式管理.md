>  **本文已授权「玉刚说」微信公众号独家发布**

### 2019/12/24 补充
距本文发布时隔一年，笔者认为，本文不应该作为入门教程的博客系列，相反，读者真正想要理解 Paging 的使用，应该先尝试理解其分页组件的本质思想：

>  [反思|Android 列表分页组件Paging的设计与实现：系统概述](https://juejin.im/post/5db06bb6518825646d79070b)  
[反思|Android 列表分页组件Paging的设计与实现：架构设计与原理解析](https://juejin.im/post/5de3df466fb9a07161484030)

以上两篇文章将对Paging分页组件进行了系统性的概述，笔者强烈建议 读者将以上两篇文章作为学习 Paging 阅读优先级 **最高** 的学习资料，**所有其它的Paging中文博客阅读优先级都应该靠后**。

本文及相关引申阅读：

> [Android官方架构组件Paging：分页库的设计美学](https://juejin.im/post/5c53ad9e6fb9a049eb3c5cfd)  
 [Android官方架构组件Paging-Ex：为分页列表添加Header和Footer](https://juejin.im/post/5caa0052f265da24ea7d3c2c)  
 [Android官方架构组件Paging-Ex：列表状态的响应式管理](https://juejin.im/post/5ce6ba09e51d4555e372a562)

## 概述

`Paging`是`Google`在2018年I/O大会上推出的适用于`Android`原生开发的分页库，随着越来越多的开发者着手使用`Paging`，越来越多的问题暴露出来，最直接的一个问题是：

> 如何管理列表额外的状态？

这样的需求随处可见，比如 `侧滑删除`、`为评论点赞` 等等：

本文将阐述：如何管理`Paging`分页列表的 **状态**，为何这样设计，以及设计的过程。

## 列表的状态问题

和市面上其它热门的分页库相比，`Paging`最大的亮点在于其 **将列表分页加载的逻辑作为回调函数封装入 `DataSource` 中**，开发者在配置完成后，无需通过代码手动控制分页的加载，列表会 **自动加载** 下一页数据并展示。

这种便利意味着开发者不需要自己持有 **数据源** ，大多数时候这使得开发流程更加便利，但总有偶然，比如这样一个界面：

![](https://raw.githubusercontent.com/qingmei2/qingmei2-blogs-art/master/android/jetpack/paging/ex2-state/1.lvuyq8lwlyp.gif)

这种需求屡见不鲜，其本质是，列表本身展示服务端返回的列表数据之外，还需要 **本地控制额外的状态**。

什么叫 **额外的状态** ? 我们先用简单的一张图展示没有额外状态的情形，这时，列表的所有UI元素都从服务端获取：

![](https://raw.githubusercontent.com/qingmei2/qingmei2-blogs-art/master/android/jetpack/paging/ex2-state/image.bkcjbsiq9b6.png)

现在我们将上文`Gif`中的点赞效果也通过一张图表示：

![](https://raw.githubusercontent.com/qingmei2/qingmei2-blogs-art/master/android/jetpack/paging/ex2-state/image.1994ir2uyn8.png)

读者可能还未认识到两种业务场景之间的差异性：对于列表的初始化来讲，所有UI元素都被服务端返回的数据渲染，每条评论是否已经被点赞，服务端都通过`Comment`进行了描述。

需要注意的是，在某一刻，用户发现某个评论非常有趣，因此他选择对该评论进行了点赞的操作。

在业务代码中，我们需要向服务端`POST`一个点赞的请求，服务端返回了一个200的成功码，但问题来了，接下来我们 **如何让列表中的那条评论状态发生变化**（即点赞的icon由灰色变成绿色高亮，已告知用户点赞成功）？

这就引发了文章最开始的那个问题，**当列表的状态发生了变更，如何管理并更新列表？**

### 方案1：再次刷新请求接口

最简单的方案是再次请求API，每当列表状态发生了变更，重新拉取评论列表，服务端返回的最新数据中，该评论自然已经被点赞了（即列表正确进行了更新）。

读者应该清楚，该方案实际并不可行，原因有二：

* **成本太高**：某些操作对于用户来说，应该是非常 **轻量级** 的（比如点赞），他们甚至希望这些操作能够 **立即被响应** 在UI上 ，而请求API并刷新列表这一个过程太重了，即使不考虑服务器的负担，对于用户来说，UI的刷新需要数秒的等待也是非常糟糕的体验。  
* **不符合逻辑**：我们更需要注意的是，`Paging`是一个分页列表，而刷新请求行为对于分页列表来说，是一个不符合产品预期的行为（比如，我的点赞操作是针对第5页的某个评论执行的，产品的设计不可能允许每次点赞都重置为列表的第一页数据，这意味着极度糟糕的用户体验）。

现在我们理解了 **每当列表状态发生了变更就刷新接口** 并非良策，因为这种通过 **远程重新拉取数据源** 更新UI的方式成本太高了。

### 方案2：额外维护一个状态的列表

大概思路是在内存中为`RecyclerView`维护一个额外的`List`，用于一一映射对应`position`的`Item`状态：

```Kotlin
class CommentPagedAdapter(
  private val likedList: ArrayList<Boolean>
)
```

通过在内存中维护这样一个`List`，的确可以实现需求，但读者需要认识到的是，`Paging`分页库本身最大的优点便是 **随着列表的滚动自动加载分页数据**，每次分页的行为开发者并不需要手动配置，并通过调用类似`notifyItemRangeInserted()`的方法更新UI。

很显然，每当分页数据获取后，开发者依然需要手动维护这个额外状态的`List`——方案2和选择使用`Paging`的初衷背道而驰，因此它并非最优先考虑的方案。

### 库本身设计的问题？

现在问题是，既不能通过 **服务端** 作为数据源，也不能在 **内存中** 额外维护一个状态的列表， 读者难免会质疑`Paging`库本身设计的问题。

> 我该如何控制列表额外的状态（包括修改、增加或者删除）？

事实上该问题已经在Github的这个 [issue](https://github.com/googlesamples/android-architecture-components/issues/281)  中进行了讨论，`Google`的工程师的回复是：

> 从技术的角度而言 ,我们可以创建一个允许部分更改数据源的API，但之后我们需要记录这些改动并在主线程上重新传递给列表。这种方法的问题在于，如果你有一个已停止的`RecyclerView`（也就是后堆栈），它将不会（也不应该）接收任何更新，因此`PagedList`将保留这个可能很长的数据列表并重新应用于主线程上的每个观察者。

> **这使问题变得非常复杂，这就是我们使用单个列表的原因。**

显然，`Paging`考虑到了更多，和市面上 **什么都能做** 的框架相比，它 **敢于收紧开发者API的调用权限**，在开发者们发挥更多奇思妙想之前，将其紧紧束缚到了可控制的范围之内，这也是笔者非常推崇`Paging`的原因之一。

 那么我们该如何处理我们的业务？此时引入一个新的角色似乎是一个不错的选择，那就是 **持久层**（即缓存）。

## 通过架构解决业务问题

综上所述，对于分页列表的状态管理问题，需要做到的是：

* 1.将一个单独的`List`交给`Paging`去进行分页加载并渲染（不应在内存中手动维护一个额外状态的列表）;  
* 2.不应该每次都通过重的操作刷新数据源（比如网络请求刷新接口）。

因此，我们需要一个 **中间件** 进行业务的调度——在需要刷新整个数据源的时候（比如用户的下拉刷新操作），从服务端拉取数据；在不需要繁重的操作时（比如用户针对某个评论进行点赞），仅仅需要针对单个数据源进行简单的修改。

这已经不单单是业务业务的问题，并且涉及到了项目本身的架构，接下来， **持久层** （即本地缓存）闪亮登场。

### 1.用持久层作为唯一的数据源

> `Android`平台的数据库框架有很多种，本文以官方的架构组件`Room`为例。

为什么要为项目的架构额外添加一个持久层？事实上，随着项目体系的日益庞大，数据库是终究需要添加进入项目中的，因此，在设计项目的架构之前，提前将数据库的框架配置进来是一个不错的选择——未雨绸缪总不是坏事。

以列表的渲染为例，让我们来看看项目之前的结构：

![](https://raw.githubusercontent.com/qingmei2/qingmei2-blogs-art/master/android/jetpack/paging/ex2-state/image.chn5qnhpz4t.png)

回到本文，对于`Paging`来讲，我们并无法直接获取数据源，因此对于列表状态的管理，我们需要额外的角色帮助，那就是本地的持久化缓存。

让我们看看添加了持久层之后的结构：

![](https://raw.githubusercontent.com/qingmei2/qingmei2-blogs-art/master/android/jetpack/paging/ex2-state/image.qtmxjksba9.png)

添加了缓存之后，每当我们尝试初始化一个分页列表，框架会从服务器拉取数据，之后数据被存储到了`Room`中。

请注意！**`Paging`原生提供了对`Room`数据库框架的支持，因此它总是可以第一时间响应到数据库中数据的变化，并自动渲染在UI上**。

现在，我们将 **请求服务器API** 和 **数据的渲染** 两者通过持久层进行了隔离，对于`RecyclerView`来说，持久层是唯一的数据源，即：

> 列表只反应了数据库的变更。

现在列表的显示和服务端的请求已经 **完全无关** 了，读者也许会有这样的疑问——这样做的好处是什么？

### 2.列表状态的管理

现在我们回到文中最初的问题，如何管理列表的状态？

对于一个拥有复杂状态的分页列表，无论是 **服务端** 作为数据源，还是在 **内存中** 额外维护一个状态列表，都不是很好的选择；而现在我们加入了`Room`，并作为列表唯一的数据源，局势发生了怎样微妙的变化呢？

让我们来看看加入了持久层之后，下拉刷新的逻辑发生了怎样的变化：

![](https://raw.githubusercontent.com/qingmei2/qingmei2-blogs-art/master/android/jetpack/paging/ex2-state/image.ukynnai7mab.png)

* 1.下拉刷新意味着我们需要重置数据，因此我们手动清除了数据库内对应表中的数据；  
* 2.当表中数据被清空时，`Paging`会自动响应到数据的变化，因为没有了数据，所以`Paging`会自动向服务器请求数据；  
* 3.数据返回后，会再次将数据存储到数据库中；  
* 4.这时`Paging`会再次响应到数据库的变化，并将最新的数据渲染到UI上。

看起来逻辑复杂了很多，实际上读者需要明确的是，步骤2、3、4都是我们作为开发者在初始化`Paging`时就配置好的，因此如果用户需要刷新页面，只需要进行第一步的操作即可，即类似这样的一行代码：

```Kotlin
// 刷新操作，仅需清除表内的列表数据
fun swipeRefresh() {
  // 运行一个事务
  db.runInTransaction {
      // 清除列表数据
      db.getDao().clearDataList()
  }
}
```

现在我们将整个流程中，`Paging`自动执行的步骤用紫色标记出来：

![](https://raw.githubusercontent.com/qingmei2/qingmei2-blogs-art/master/android/jetpack/paging/ex2-state/image.5x125xqmmvp.png)

瞧，除了我们手动执行的逻辑，所有流程都交给了`Paging`去 **响应式** 地执行。

我们总是下意识认为复杂的业务逻辑用过程式的编码更容易实现，`Paging`用事实证明了并非如此——如果说项目中的某个页面追加了下拉刷新的需求，过程式的编码也许会花费更多的时间，并且代码也许会更分散、啰嗦且易出错。

### 3.更灵活、且可高度扩展

接下来分析的是，对分页列表点赞这种相对 **轻量级的行为** 又该如何处理?

答案呼之欲出, 我们依然用熟悉的流程图表示代码的执行步骤：

![](https://raw.githubusercontent.com/qingmei2/qingmei2-blogs-art/master/android/jetpack/paging/ex2-state/image.wbm75ah4osa.png)

即使是复杂的状态，在这种模式下也不再是难题：首先，我们将数据库对应表中对应评论的`isLike`（是否被点赞）设置为`true`：

```Kotlin
// 1.对本地的评论数据点赞
fun likeCommentLocal(comment: Comment) {
  // 更新评论
  comment.isLike = true
  // 将评论更新到数据库中
  db.runInTransaction {
     db.getDao().updateLikeComment(o)
  }
}
```

与此同时，我们也向服务器请求接口，告知评论被用户点赞：

```Kotlin
// 2.对评论点赞
fun likeCommentRemote(commentId: String) {
  service.likeComment(commentId)
  // ....
}
```

当数据库中数据发生了变更，`Paging`仍然会响应到数据的更新，并第一时间更新了UI，同时我们也向服务器发起了请求，一个完整的 **点赞** 操作相关的业务代码实现完毕。

有了持久层作为中间件，**代码组织的灵活性大大提升，同时也具备了更高的扩展性**。列表状态的管理不再是问题，诸如 **点赞** 、 **下拉刷新** 、 **侧滑删除** 等等等等，都可以通过对持久层的数据源进行修改，`paging`总是可以第一时间自动响应到变更并更新UI。

也正如`Room`官方文档第一句话所说的，对于`Paging`分页列表（对app也一样）复杂的状态的展示和管理，开发者应该 **将缓存作为列表的唯一真实的数据源**：

> This cache, which serves as your app's single source of truth.

## 代码示例？

如读者所看到的，本文尽量避免展示大篇幅的业务代码，原因有二：

* 1.这会破坏文章整体思路的完整性，没有人喜欢阅读大篇幅、连续的代码片段；
* 2.实际开发中，项目的业务不同、架构选型不同，代码的实现方式也不尽相同，因此业务级别的代码展示没有意义。

> 比如，对于持久层框架的选型，`Room`、`GreenDao`、`DBFlow`都是非常优秀的框架，对于业务代码的实现，`RxJava`、`LiveData`、协程都是优秀的实现方案...

本文的目的是阐述笔者遇到问题的解决步骤和思路，读者了解整体的方案之后，可以根据实际项目进行技术选型。

当然，如果有相关的疑惑，欢迎参考下面两个项目的具体实现，这是笔者基于上文的`Paging`+`Room`组件，实现了一个简单的`Github`的客户端，本文不细述。

1.`MVVM`架构的Sample: https://github.com/qingmei2/MVVM-Rhine

2.`MVI`架构的Sample:https://github.com/qingmei2/MVI-Rhine

## 系列文章

>  **争取打造 Android Jetpack 讲解的最好的博客系列**：
>* [Android官方架构组件Lifecycle：生命周期组件详解&原理分析](https://juejin.im/post/5c53beaf51882562e27e5ad9)
>* [Android官方架构组件ViewModel:从前世今生到追本溯源](https://juejin.im/post/5c047fd3e51d45666017ff86)
>* [Android官方架构组件LiveData: 观察者模式领域二三事](https://juejin.im/post/5c25753af265da61561f5335)
>* [Android官方架构组件Paging：分页库的设计美学](https://juejin.im/post/5c53ad9e6fb9a049eb3c5cfd)
>* [Android官方架构组件Paging-Ex：为分页列表添加Header和Footer](https://juejin.im/post/5caa0052f265da24ea7d3c2c)
>* [Android官方架构组件Paging-Ex：列表状态的响应式管理](https://juejin.im/post/5ce6ba09e51d4555e372a562)
>* [Android官方架构组件Navigation：大巧不工的Fragment管理框架](https://juejin.im/post/5c53be3951882562d27416c6)  
>* [Android官方架构组件DataBinding-Ex:双向绑定篇](https://juejin.im/post/5c3e04b7f265da611b589574)  

> **Android Jetpack 实战篇**：
>* [开源项目：MVVM+Jetpack实现的Github客户端](https://github.com/qingmei2/MVVM-Rhine)
>* [开源项目：基于MVVM, MVI+Jetpack实现的Github客户端](https://github.com/qingmei2/MVI-Rhine)
>* [总结：使用MVVM尝试开发Github客户端及对编程的一些思考](https://juejin.im/post/5be7bbd9f265da61797458cf)

---

## 关于我

Hello，我是[却把清梅嗅](https://github.com/qingmei2)，如果您觉得文章对您有价值，欢迎 ❤️，也欢迎关注我的[博客](https://www.jianshu.com/u/df76f81fe3ff)或者[Github](https://github.com/qingmei2)。

如果您觉得文章还差了那么点东西，也请通过**关注**督促我写出更好的文章——万一哪天我进步了呢？

* [我的Android学习体系](https://github.com/qingmei2/android-programming-profile)
* [关于文章纠错](https://github.com/qingmei2/Programming-life/blob/master/error_collection.md)
* [关于知识付费](https://github.com/qingmei2/Programming-life/blob/master/appreciation.md)
