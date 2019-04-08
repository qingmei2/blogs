# Android官方架构组件Paging-Ex:为分页列表添加Header和Footer

## 概述

`Paging`是`Google`在2018年I/O大会上推出的适用于`Android`原生开发的分页库，如果您还不是很了解这个 **官方钦定** 的分页架构组件，欢迎参考笔者的这篇文章：

https://mp.weixin.qq.com/s?__biz=MzIwMTAzMTMxMg==&mid=2649492903&idx=1&sn=6040b030d2a8125f38b7c9e7bd8f3054&chksm=8eec8658b99b0f4e07cf1c550096b87c5551a6da124906a67911dcae4a0656814a6e5cc13c84#rd

笔者在实际项目中已经使用`Paging`半年有余，和市面上其它热门的分页库相比，`Paging`最大的亮点在于其 **将列表分页加载的逻辑作为回调函数封装入 `DataSource` 中**，开发者在配置完成后，无需控制分页的加载，列表会 **自动加载** 下一页数据并展示。

本文将阐述：为使用了`Paging`的列表添加`Header`和`Footer`的整个过程、这个过程中遇到的一些阻碍、以及自己是如何解决这些阻碍的——如果您想直接浏览最终的解决方案，请直接翻阅本文的 **最终的解决方案** 小节。

## 初始思路

为`RecyclerView`列表添加`Header`或`Footer`并不是一个很麻烦的事，最简单粗暴的方式是将`RecyclerView`和`Header`共同放入同一个`ScrollView`的子`View`中，但它无异于对`RecyclerView`自身的复用机制视而不见，因此这种解决方案并非首选。

更适用的解决方式是通过实现 **多类型列表**（MultiType），除了列表本身的`Item`类型之外，`Header`或`Footer`也被视作一种`Item`，关于这种方式的实现网上已有很多文章讲解，本文不赘述。

在正式开始本文内容之前，我们先来看看最终的实现效果，我们为一个`Student`的分页列表添加了一个`Header`和`Footer`：

![](https://user-gold-cdn.xitu.io/2019/4/7/169f81151ef99c66?w=524&h=927&f=gif&s=495280)

实现这种效果，笔者最初的思路也是通过 **多类型列表** 实现`Header`和`Footer`，但是很快我们就遇到了第一个问题，那就是 **我们并没有直接持有数据源**。

## 1.数据源问题

对于常规的多类型列表而言，我们可以轻易的持有`List<ItemData>`，从数据的控制而言，我更倾向于用一个代表`Header`或者`Footer`的占位符插入到数据列表的顶部或者底部，这样对于`RecyclerView`的渲染而言，它是这样的：

![](https://user-gold-cdn.xitu.io/2019/4/7/169f81151ef4e8fb?w=688&h=558&f=png&s=31574)

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

## 2.尝试直接实现列表

接下来我针对直接实现多类型列表进行尝试，我们先不讨论如何实现`Footer`，仅以`Header`而言，我们进行如下的实现：

```Kotlin
class HeaderSimpleAdapter : PagedListAdapter<Student, RecyclerView.ViewHolder>(diffCallback) {

    // 1.根据position为item分配类型
    // 如果position = 1，视为Header
    // 如果position != 1,视为普通的Student
    override fun getItemViewType(position: Int): Int {
        return when (position == 0) {
            true -> ITEM_TYPE_HEADER
            false -> super.getItemViewType(position)
        }
    }

    // 2.根据不同的viewType生成对应ViewHolder
    override fun onCreateViewHolder(parent: ViewGroup, viewType: Int): RecyclerView.ViewHolder {
        return when (viewType) {
            ITEM_TYPE_HEADER -> HeaderViewHolder(parent)
            else -> StudentViewHolder(parent)
        }
    }

    // 3.根据holder类型，进行对应的渲染
    override fun onBindViewHolder(holder: RecyclerView.ViewHolder, position: Int) {
        when (holder) {
            is HeaderViewHolder -> holder.renderHeader()
            is StudentViewHolder -> holder.renderStudent(getStudentItem(position))
        }
    }

    // 4.这里我们根据StudentItem的position，
    // 获取position-1位置的学生
    private fun getStudentItem(position: Int): Student? {
        return getItem(position - 1)
    }

    // 5.因为有Header，item数量要多一个
    override fun getItemCount(): Int {
        return super.getItemCount() + 1
    }

    // 省略其他代码...
    // 省略ViewHolder代码
}    
```

代码和注释已经将我的个人思想展示的很清楚了，我们固定一个`Header`在多类型列表的最上方，这也导致我们需要重写`getItemCount()`方法，并且在对`Item`进行渲染的`onBindViewHolder()`方法中，对`Sutdent`的获取进行额外的处理——因为多了一个Header，导致产生了数据源和列表的错位差—— **第n个数据被获取时，我们应该将其渲染在列表的第n+1个位置上**。

我简单绘制了一张图来描述这个过程，也许更加直观易懂：

![](https://user-gold-cdn.xitu.io/2019/4/7/169f81151ee1ecbb?w=738&h=566&f=png&s=47854)

代码写完后，直觉告诉我似乎没有什么问题，让我们来看看实际的运行效果：

![](https://user-gold-cdn.xitu.io/2019/4/7/169f81151ebcb06b?w=518&h=916&f=gif&s=1230516)

Gif也许展示并不那么清晰，简单总结下，问题有两个：

* 1.在我们进行下拉刷新时，因为`Header`更应该是一个**静态独立的组件**，但实际上它也是列表的一部分，因此白光一闪，除了`Student`列表，`Header`作为`Item`也进行了刷新，这与我们的预期不符；
* 2.下拉刷新之后，列表 **并未展示在最顶部**，而是滑动到了一个奇怪的位置。

导致这两个问题的根本原因仍然是`Paging`计算列表的`position`时出现的问题:

对于问题1，`Paging`对于列表的刷新理解为 **所有Item的刷新**，因此同样作为`Item`的`Header`也无法避免被刷新；

问题2依然也是这个问题导致的，在`Paging`获取到第一页数据时（假设第一页数据只有10条），`Paging`会命令更新`position in 0..9`的`Item`,而实际上因为`Header`的关系，我们是期望它能够更新第`position in 1..10`的`Item`，最终导致了刷新以及对新数据的展示出现了问题。

## 3.向Google和Github寻求答案

正如标题而言，我尝试求助于`Google`和`Github`，最终找到了这个链接：

https://github.com/googlesamples/android-architecture-components/issues/548

如果您简单研究过`PagedListAdapter`的源码的话，您应该了解，`PagedListAdapter`内部定义了一个`AsyncPagedListDiffer`，用于对列表数据的加载和展示，`PagedListAdapter`更像是一个空壳，所有分页相关的逻辑实际都 **委托** 给了`AsyncPagedListDiffer`:

```Java
public abstract class PagedListAdapter<T, VH extends RecyclerView.ViewHolder>
        extends RecyclerView.Adapter<VH> {

         final AsyncPagedListDiffer<T> mDiffer;

         public void submitList(@Nullable PagedList<T> pagedList) {
             mDiffer.submitList(pagedList);
         }

         protected T getItem(int position) {
             return mDiffer.getItem(position);
         }

         public int getItemCount() {
             return mDiffer.getItemCount();
         }       

         public PagedList<T> getCurrentList() {
             return mDiffer.getCurrentList();
         }
}          
```

虽然`Paging`中数据的获取和展示我们是无法控制的，但我们可以尝试 **瞒过** `PagedListAdapter`,即使`Paging`得到了`position in 0..9`的`List<Data>`,但是我们让`PagedListAdapter`去更新`position in 1..10`的item不就可以了嘛？

因此在上方的`Issue`链接中，**[onlymash](https://github.com/onlymash)** 同学提出了一个解决方案：

**重写`PagedListAdapter`中被`AsyncPagedListDiffer`代理的所有方法，然后实例化一个新的`AsyncPagedListDiffer`，并让这个新的differ代理这些方法。**

篇幅所限，我们只展示部分核心代码：

```Kotlin
class PostAdapter: PagedListAdapter<Any, RecyclerView.ViewHolder>() {

    private val adapterCallback = AdapterListUpdateCallback(this)

    // 当第n个数据被获取，更新第n+1个position
    private val listUpdateCallback = object : ListUpdateCallback {
        override fun onChanged(position: Int, count: Int, payload: Any?) {
            adapterCallback.onChanged(position + 1, count, payload)
        }

        override fun onMoved(fromPosition: Int, toPosition: Int) {
            adapterCallback.onMoved(fromPosition + 1, toPosition + 1)
        }

        override fun onInserted(position: Int, count: Int) {
            adapterCallback.onInserted(position + 1, count)
        }

        override fun onRemoved(position: Int, count: Int) {
            adapterCallback.onRemoved(position + 1, count)
        }
    }

    // 新建一个differ
    private val differ = AsyncPagedListDiffer<Any>(listUpdateCallback,
        AsyncDifferConfig.Builder<Any>(POST_COMPARATOR).build())

    // 将所有方法重写，并委托给新的differ去处理
    override fun getItem(position: Int): Any? {
        return differ.getItem(position - 1)
    }

    // 将所有方法重写，并委托给新的differ去处理
    override fun submitList(pagedList: PagedList<Any>?) {
        differ.submitList(pagedList)
    }

    // 将所有方法重写，并委托给新的differ去处理
    override fun getCurrentList(): PagedList<Any>? {
        return differ.currentList
    }
}
```

现在我们成功实现了上文中我们的思路，一图胜千言：

![](https://user-gold-cdn.xitu.io/2019/4/7/169f81151ee1ecbb?w=738&h=566&f=png&s=47854)

## 4.另外一种实现方式

上一小节的实现方案是完全可行的，但我个人认为美中不足的是，这种方案 **对既有的`Adapter`中代码改动过大**。

我新建了一个`AdapterListUpdateCallback`、一个`ListUpdateCallback`以及一个新的`AsyncPagedListDiffer`,并重写了太多的`PagedListAdapter`的方法——我添加了数十行分页相关的代码，但这些代码和正常的列表展示并没有直接的关系。

当然，我可以将这些逻辑都抽出来放在一个新的类里面，但我还是感觉我 **好像是模仿并重写了一个新的`PagedListAdapter`类一样**，那么是否还有其它的思路呢？

最终我找到了这篇文章:

https://blog.csdn.net/cekiasoo/article/details/81990475

这篇文章中的作者通过细致分析`Paging`的源码，得出了一个更简单实现`Header`的方案，有兴趣的同学可以点进去查看，这里简单阐述其原理：

通过查看源码，以添加分页为例，**`Paging`对拿到最新的数据后，对列表的更新实际是调用了`RecyclerView.Adapter`的`notifyItemRangeInserted()`方法,而我们可以通过重写`Adapter.registerAdapterDataObserver()`方法，对数据更新的逻辑进行调整**：


```Java
// 1.新建一个 AdapterDataObserverProxy 类继承 RecyclerView.AdapterDataObserver
class AdapterDataObserverProxy extends RecyclerView.AdapterDataObserver {
    RecyclerView.AdapterDataObserver adapterDataObserver;
    int headerCount;
    public ArticleDataObserver(RecyclerView.AdapterDataObserver adapterDataObserver, int headerCount) {
        this.adapterDataObserver = adapterDataObserver;
        this.headerCount = headerCount;
    }
    @Override
    public void onChanged() {
        adapterDataObserver.onChanged();
    }
    @Override
    public void onItemRangeChanged(int positionStart, int itemCount) {
        adapterDataObserver.onItemRangeChanged(positionStart + headerCount, itemCount);
    }
    @Override
    public void onItemRangeChanged(int positionStart, int itemCount, @Nullable Object payload) {
        adapterDataObserver.onItemRangeChanged(positionStart + headerCount, itemCount, payload);
    }

    // 当第n个数据被获取，更新第n+1个position
    @Override
    public void onItemRangeInserted(int positionStart, int itemCount) {
        adapterDataObserver.onItemRangeInserted(positionStart + headerCount, itemCount);
    }
    @Override
    public void onItemRangeRemoved(int positionStart, int itemCount) {
        adapterDataObserver.onItemRangeRemoved(positionStart + headerCount, itemCount);
    }
    @Override
    public void onItemRangeMoved(int fromPosition, int toPosition, int itemCount) {
        super.onItemRangeMoved(fromPosition + headerCount, toPosition + headerCount, itemCount);
    }
}

// 2.对于Adapter而言，仅需重写registerAdapterDataObserver()方法
//   然后用 AdapterDataObserverProxy 去做代理即可
class PostAdapter extends PagedListAdapter {

    @Override
    public void registerAdapterDataObserver(@NonNull RecyclerView.AdapterDataObserver observer) {
        super.registerAdapterDataObserver(new AdapterDataObserverProxy(observer, getHeaderCount()));
    }
}
```

我们将额外的逻辑抽了出来作为一个新的类，思路和上一小节的十分相似，同样我们也得到了预期的结果。

经过对源码的追踪，从性能上来讲，这两种实现方式并没有什么不同，唯一的区别就是，前者是针对`PagedListAdapter`进行了重写，将`Item`更新的代码交给了`AsyncPagedListDiffer`;而这种方式中，`AsyncPagedListDiffer`内部对`Item`更新的逻辑，最终仍然是交给了`RecyclerView.Adapter`的`notifyItemRangeInserted()`方法去执行的——**两者本质上并无区别**。

## 5.最终的解决方案

虽然上文只阐述了`Paging library`如何实现`Header`，实际上对于`Footer`而言也是一样，因为`Footer`也可以被视为另外一种的`Item`；同时，因为`Footer`在列表底部，并不会影响`position`的更新，因此它更简单。

下面是完整的`Adapter`示例：

```kotlin
class HeaderProxyAdapter : PagedListAdapter<Student, RecyclerView.ViewHolder>(diffCallback) {

    override fun getItemViewType(position: Int): Int {
        return when (position) {
            0 -> ITEM_TYPE_HEADER
            itemCount - 1 -> ITEM_TYPE_FOOTER
            else -> super.getItemViewType(position)
        }
    }

    override fun onCreateViewHolder(parent: ViewGroup, viewType: Int): RecyclerView.ViewHolder {
        return when (viewType) {
            ITEM_TYPE_HEADER -> HeaderViewHolder(parent)
            ITEM_TYPE_FOOTER -> FooterViewHolder(parent)
            else -> StudentViewHolder(parent)
        }
    }

    override fun onBindViewHolder(holder: RecyclerView.ViewHolder, position: Int) {
        when (holder) {
            is HeaderViewHolder -> holder.bindsHeader()
            is FooterViewHolder -> holder.bindsFooter()
            is StudentViewHolder -> holder.bindTo(getStudentItem(position))
        }
    }

    private fun getStudentItem(position: Int): Student? {
        return getItem(position - 1)
    }

    override fun getItemCount(): Int {
        return super.getItemCount() + 2
    }

    override fun registerAdapterDataObserver(observer: RecyclerView.AdapterDataObserver) {
        super.registerAdapterDataObserver(AdapterDataObserverProxy(observer, 1))
    }

    companion object {
        private val diffCallback = object : DiffUtil.ItemCallback<Student>() {
            override fun areItemsTheSame(oldItem: Student, newItem: Student): Boolean =
                    oldItem.id == newItem.id

            override fun areContentsTheSame(oldItem: Student, newItem: Student): Boolean =
                    oldItem == newItem
        }

        private const val ITEM_TYPE_HEADER = 99
        private const val ITEM_TYPE_FOOTER = 100
    }
}
```
如果你想查看运行完整的demo，这里是本文sample的地址：

> https://github.com/qingmei2/SamplePaging

## 6.更多优化点？

文末最终的方案是否有更多优化的空间呢？当然，在实际的项目中，对其进行简单的封装是更有意义的（比如`Builder`模式、封装一个`Header`、`Footer`甚至两者都有的装饰器类、或者其它...）。

本文旨在描述Paging使用过程中 **遇到的问题** 和 **解决问题的过程**，因此项目级别的封装和实现细节不作为本文的主要内容；关于`Header`和`Footer`在`Paging`中的实现方式，如果您有更好的解决方案，欢迎提出，期待与您的共同探讨。

## 参考&感谢

* Paging library 源码
* Android RecyclerView + Paging Library 添加头部刷新会自动滚动的问题分析及解决
https://blog.csdn.net/cekiasoo/article/details/81990475
* PagingWithNetworkSample - PagedList RecyclerView scroll bug
https://github.com/googlesamples/android-architecture-components/issues/548Â
