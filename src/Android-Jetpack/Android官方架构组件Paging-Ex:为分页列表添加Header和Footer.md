# Android官方架构组件Paging-Ex:为分页列表添加Header和Footer

## 概述

`Paging`是`Google`在2018年I/O大会上推出的适用于`Android`原生开发的分页库，如果您还不是很了解这个 **官方钦定** 的分页架构组件，欢迎参考笔者的这篇文章：

>* [Android官方架构组件Paging：分页库的设计美学](https://www.jianshu.com/p/10bf4bf59122)

笔者在实际项目中已经使用`Paging`半年有余，和市面上其它热门的分页库相比，`Paging`最大的亮点在于其 **将列表分页加载的逻辑作为回调函数封装入 `DataSource` 中**，开发者在配置完成后，无需控制分页的加载，列表会 **自动加载** 下一页数据并展示。

本文将阐述：为使用了`Paging`的列表添加`Header`和`Footer`的整个过程、这个过程中遇到的一些阻碍、以及自己是如何解决这些阻碍的——如果您想直接浏览最终的解决方案，请直接翻阅本文的 **最终的解决方案** 小节。

## 初始思路

为`RecyclerView`列表添加`Header`或`Footer`并不是一个很麻烦的事，最简单粗暴的方式是将`RecyclerView`和`Header`共同放入同一个`ScrollView`的子`View`中，但它无异于对`RecyclerView`自身的复用机制视而不见，因此这种解决方案并非首选。

更适用的解决方式是通过实现 **多类型列表**（MultiType），除了列表本身的`Item`类型之外，`Header`或`Footer`也被视作一种`Item`，关于这种方式的实现网上已有很多文章讲解，本文不赘述。

在正式开始本文内容之前，我们先来看看最终的实现效果，我们为一个`Student`的分页列表添加了一个`Header`和`Footer`：



实现这种效果，笔者最初的思路也是通过 **多类型列表** 实现`Header`和`Footer`，但是很快我们就遇到了第一个问题，那就是 **我们并没有直接持有数据源**。

## 1.数据源问题

对于常规的多类型列表而言，我们可以轻易的持有`List<ItemData>`，从数据的控制而言，我更倾向于用一个代表`Header`或者`Footer`的占位符插入到数据列表的顶部或者底部，这样对于`RecyclerView`的渲染而言，它是这样的：


正如我所标注的，`List<ItemData>`中一个`ItemData`对应了一个`ItemView`——我认为为一个`Header`或者`Footer`单独创建对应一个`Model`类型是完全值得的，它极大增强了代码的可读性，而且对于复杂的`Header`而言，代表状态的`Model`类也更容易让开发者对其进行渲染。

这种实现方式简单、易读而不失优雅，但是在`Paging`中，这种思路一开始就被堵死了。

我们先看`PagedListAdapter`类的声明：

```Java
// T泛型代表数据源的类型，即本文中的 Student
public abstract class PagedListAdapter<T, VH extends RecyclerView.ViewHolder>
      extends RecyclerView.Adapter<VH> {
    // ...
}
```

因此，我们需要这样实现：

```kotlin
// 这里我们只能指定Student类型
class SimpleAdapter : PagedListAdapter<Student, RecyclerView.ViewHolder>(diffCallback) {
  // ...
}
```

有同学提出，我们可以将这里的`Student`指定为某个接口（比如定义一个`ItemData`接口），然后让`Student`和`Header`对应的`Model`都去实现这个接口，然后这样：

```kotlin
class SimpleAdapter : PagedListAdapter<ItemData, RecyclerView.ViewHolder>(diffCallback) {
  // ...
}
```

看起来确实可行，但是我们忽略了一个问题，那就是本小节要阐述的：

> **我们并没有直接持有数据源**。

回到初衷，我们知道，`Paging`最大的亮点在于 **自动分页加载**，这是观察者模式的体现，配置完成后，我们并不关心 **数据是如何被分页、何时被加载、如何被渲染** 的，因此我们也不需要直接持有`List<Student>`（实际上也持有不了）,更无从谈起手动为其添加`HeaderItem`和`FooterItem`了。

以本文为例，实际上所有逻辑都交给了`ViewModel`：

```KOTLIN
class CommonViewModel(app: Application) : AndroidViewModel(app) {

    private val dao = StudentDb.get(app).studentDao()

    fun getRefreshLiveData(): LiveData<PagedList<Student>> =
            LivePagedListBuilder(dao.getAllStudent(), PagedList.Config.Builder()
                    .setPageSize(15)                         //配置分页加载的数量
                    .setInitialLoadSizeHint(40)              //初始化加载的数量
                    .build()).build()
}
```

可以看到，我们并未直接持有`List<Student>`，因此`list.add(headerItem)`这种 **持有并修改数据源** 的方案几乎不可行（较真而言，其实是可行的，但是成本过高，本文不深入讨论）。
