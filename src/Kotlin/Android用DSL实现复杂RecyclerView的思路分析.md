> 阅读本文需要您对 **DSL, Kotlin, DataBinding** 有一定的了解,阅读时长约 **8分钟**。

## ReactiveX

**响应式编程**是一种面向数据流和变化传播的编程范式。随着自己知识领域的逐渐深入，我越来越依赖 **RxJava**。在Java语言中，通过lambda和方法引用，配合**RxJava**额外提供的**函数式接口**，链式调用的代码写起来既**优美**又有**逼格**。

**RxJava**的毒性令人欲罢不能，我趋之若鹜，尝试在自己的 **[Github](https://github.com/qingmei2)** 上做出了很多Rx相关工具的尝试（比如[这个](https://github.com/qingmei2/RxImagePicker)）。不久前，在 [Yumenokanata](https://github.com/Yumenokanata) 的影响下，我尝试实现一个基于RxJava的Dialog库 [RxDialog](https://github.com/qingmei2/RxDialog)，以对应原生Dialog的一些行为（创建与显示，以及事件监听等等）。

## 困惑

很快，我遇到的问题是，我无法控制库对Dialog控制的**粒度**。举例来说，目前[RxDialog](https://github.com/qingmei2/RxDialog)的效果是，通过配置一些简单的参数，用Kotlin不同于Java的Builder模式，创建出一个标准的AlertDialog，并展示在界面上，而Dialog中不同的事件（点击事件，dismiss事件）则作为流被Observable通知给观察者：

```kotlin
button.setOnClickListener {
         RxDialog
                .build(this) {
                    title = "I am title"
                    message = "I am message"
                    buttons = arrayOf(
                            EventType.CALLBACK_TYPE_POSITIVE,
                            EventType.CALLBACK_TYPE_NEGATIVE,
                            EventType.CALLBACK_TYPE_DISMISS
                    )
                    positiveText = getString(R.string.static_dialog_button_ok)
                    positiveTextColor = R.color.positive_color
                    negativeText = getString(R.string.static_dialog_button_cancel)
                    negativeTextColor = R.color.negative_color
                    cancelable = false
                }
                .subscribe { event ->
                      when (event.button) {
                          EventType.CALLBACK_TYPE_POSITIVE -> {
                              toast("click the OK")
                          }
                          EventType.CALLBACK_TYPE_NEGATIVE -> {
                              toast("click the CANCEL")
                          }
                          EventType.CALLBACK_TYPE_DISMISS -> {
                              toast("dismiss...")
                          }
                      }
                }
}
```

看起来不错，作为一个工具类还凑活，但是如果面对一些自定义UI的Dialog，[RxDialog](https://github.com/qingmei2/RxDialog)就无能为力了（尴尬的是，对于产品而言，这种情况几乎是必然发生）。

同样，如果这个Dialog的内容是一个动态的列表时，RxDialog依然束手无策，简单的策略是像原生API一样，通过 AlertDialog.Builder(context).setAdapter()来实例化一个列表Dialog，但是原生的API内部实际上是实例化了一个ListView而不是RecyclerView，我不是很想这样做。

我不知道该怎么把握这个库对于Dialog UI控制的粒度，于是这个库停工了。在重新启动之前，我准备寻求一些优秀源码，从它们的设计思想中获取帮助。

RecyclerView似乎和Dialog很相似，我注意到了这样一个工具库——
[DslAdapter](https://github.com/Yumenokanata/DslAdapter)，它是上文说到的，[Yumenokanata](https://github.com/Yumenokanata) 自己写的一个工具。

## DslAdapter

> **DslAdapter**:A RecyclerView Adapter builder by DSL. Easy to use, and all code written by kotlin.

DslAdapter是一个RecyclerView Adapter的工具库，它基于Kotlin，简单易用（老实讲，个人看来不是很好上手...），关键是，你可以用DSL的方式实现列表展示。

关于DSL的解释，请参考 [百度百科：领域特定语言](https://baike.baidu.com/item/%E9%A2%86%E5%9F%9F%E7%89%B9%E5%AE%9A%E8%AF%AD%E8%A8%80/2826893?fr=aladdin)，本文不细述。

对于官方给出的sample，它不仅支持简单的列表展示，同时支持 **各种复杂的多类型列表**（包括Header和Footer），支持**DataBinding**，支持数据驱动UI的**更新**和**自动更新**。

sample中的代码展示了如何实现一个非常复杂的多类型列表，包括DataBinding和数据更新的功能：

![sample](https://upload-images.jianshu.io/upload_images/7293029-c7a0d65e74a1affd.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

现在看来，对于复杂的列表来说，不同类型的Item的展示，如何统一交给RecyclerView去展示，似乎和我遇到不同类型Dialog展示的问题非常相似，我决定尝试理解DslAdapter的设计思路，看看能不能给我带来一些好的想法。

> 本文不是教程，感兴趣的朋友请以官方sample为准——sample中的代码已经讲述的很清晰了。

* * *

## 思路分析

### 1.RendererAdapter及其Builder

**DslAdapter**的设计思路是，**将整个列表像搭积木一样组合起来**——哪怕它再复杂。对于复杂的列表而言，需要面对的就是对不同类型数据的添加，然后将不同类型的数据展示在不同的ItemLayout上。

当然，不可避免的，RecyclerView.Adapter依然是最重要的一个类，和很多开源库的思路一样，DslAdapter也同样实现了一个 **RendererAdapter**，它也是**RecyclerView.Adapter**的一个子类：

```Kotlin
class RendererAdapter(builder: Builder) :
      RecyclerView.Adapter<RecyclerView.ViewHolder>() {

}
```

RendererAdapter的实例化必须依赖一个Builder，我们来看看Builder的代码：

```Kotlin
class Builder internal constructor() {
    internal val repositories: MutableList<Repo<Any>> = ArrayList()

    fun <T, VD : ViewData> add(supplier: Supplier<T>,
                               renderer: Renderer<T, VD>): Builder {
        val untypedRepository = supplier as Supplier<Any>
        repositories.add(Repo(untypedRepository, renderer as Renderer<Any, ViewData>, false))
        return this
    }

    fun <T, VD : ViewData> addItem(item: T,
                                   renderer: Renderer<T, VD>): Builder {
        repositories.add(Repo({ item as Any }, renderer as Renderer<Any, ViewData>, true))
        return this
    }

    fun <VD : ViewData> addStaticItem(renderer: Renderer<Unit, VD>): Builder {
        repositories.add(Repo({ Unit }, renderer as Renderer<Any, ViewData>, true))
        return this
    }

    fun build(): RendererAdapter {
        return RendererAdapter(this)
    }
}

data class Repo<T>(val supplier: Supplier<T>, val renderer: Renderer<T, ViewData>, val isStatic: Boolean)
```

当一个对象的实例化需要多个参数时，建造者模式（Builder）已被认可为非常好的实现方式之一。复杂列表中的不同数据便是如此，Adapter的Builder提供了2种不同的Item接口：**静态Item** （比如Header和Footer，它们的展示不受列表数据源的变化而变化）和数据源对应的 **普通列表的Item**（一般来讲，数据源中的每一条数据对应一个Item）。

add()函数对应的是**普通列表的Item**的处理逻辑，每当有一种数据需要被展示，就调用一次add()函数，一般来讲，最简单的列表，只需要调用一次add()函数就够了。

addStaticItem()函数对应的是**静态Item**，比如说，我想添加一个Footer，只需要调用一次这个方法，作为最后一块的**积木**拼接在Adapter的最底部就行了。

当然我们看到还有一个addItem()函数，实际上它和add()函数本质是一样的，这个后文会讲到。

### 2.Supplier提供数据，Renderer处理展示数据

无论是add()函数还是addStaticItem()函数，它们涉及到了非常重要的两个参数：Supplier<T\> 和Renderer<T, VD>。

Supplier是一个函数，负责提供数据源。

```kotlin
typealias Supplier<T> = () -> T
```

Renderer则是一个接口，负责将数据响应到UI上：

```kotlin
interface Renderer<Data, VD: ViewData> {
    // 获取单个数据
    fun getData(content: Data): VD

    // 获取ItemId
    fun getItemId(data: VD, index: Int): Long = RecyclerView.NO_ID

    // 获取Item类型
    fun getItemViewType(data: VD, position: Int): Int

    // 创建ViewHolder
    fun onCreateViewHolder(parent: ViewGroup, viewType: Int): RecyclerView.ViewHolder

    // 绑定数据到UI上，你可以理解为onBindViewHolder()
    fun bind(data: VD, index: Int, holder: RecyclerView.ViewHolder)

    // View的回收
    fun recycle(holder: RecyclerView.ViewHolder)

    // 数据的更新
    fun getUpdates(oldData: VD, newData: VD): List<UpdateActions>
}

interface ViewData {
    val count: Int
}
```

第一次看到这里，我认为已经抽象的程度已经可以了，至少我研究了一段时间才缓过来。

设计者的想法是，既然RecyclerView.Adapter的作用是，获取数据，并且展示到RecyclerView上面，那么我将RecyclerView.Adapter这两个最重要的功能都抽象出来，这样一来，自己实现的DslAdapter需要做的就只剩下 **调度** 了。

当然，Renderer不止这么简单，因为根据不同的需求，有很多种不同的Renderer：

![](https://upload-images.jianshu.io/upload_images/7293029-a1b2a4d2c408a2d1.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

不同的需求采用不同的策略，也可以选择对应的Renderer，以LayoutRenderer为例：

```Kotlin
class LayoutRenderer<T : Any>(
        @LayoutRes val layout: Int,
        val count: Int = 1,
        val binder: (View, T, Int) -> Unit = { _, _, _ -> },
        val recycleFun: (View) -> Unit = { },
        val stableIdForItem: (T, Int) -> Long = { _, _ -> -1L },
        val keyGetter: (T) -> Any? = { it }
) : BaseRenderer<T, LayoutViewData<T>>() {

    override fun getData(content: T): LayoutViewData<T> = LayoutViewData(count, content)

    override fun getItemId(data: LayoutViewData<T>, index: Int): Long = stableIdForItem(data.data, index)

    override fun getLayoutResId(data: LayoutViewData<T>, index: Int): Int = layout

    override fun bind(data: LayoutViewData<T>, index: Int, holder: RecyclerView.ViewHolder) {
        binder(holder.itemView, data.data, index)
    }

    override fun recycle(holder: RecyclerView.ViewHolder) {
        recycleFun(holder.itemView)
    }

    override fun getUpdates(oldData: LayoutViewData<T>, newData: LayoutViewData<T>): List<UpdateActions> {
        if (oldData.count != newData.count)
            throw UnknownError("oldData count is different with newData count: old=$oldData, new=$newData")

        return if (keyGetter(oldData.data) != keyGetter(newData.data))
            listOf(OnRemoved(0, newData.count),
                    OnInserted(0, newData.count))
        else
            if (oldData.data != newData.data)
                listOf(OnChanged(0, newData.count, newData.data))
            else
                emptyList()
    }

    companion object
}

data class LayoutViewData<T>(override val count: Int, val data: T) : ViewData
```

实例化这样一个LayoutRenderer，需要传入很多的参数，其中大多数的参数都是函数，这就是为什么我们可以通过lambda实现这样的LayoutRenderer，并生成对应的Adapter：

```kotlin
val adapter = RendererAdapter.repositoryAdapter()
     .addStaticItem(layout<Unit>(R.layout.list_header))
     .add({ provideData(index) },
             LayoutRenderer<ItemModel>(layout = R.layout.simple_item,
                     stableIdForItem = { item, index -> item.id },
                     binder = { view, itemModel, index -> view.findViewById<TextView>(R.id.simple_text_view).text = itemModel.title },
                     recycleFun = { view -> view.findViewById<TextView>(R.id.simple_text_view).text = "" })
                     .forList({ i, index -> index }))
     .build()
```

### 3.RendererAdapter实现的细节

如果说第一眼感觉最难以理解的是Supplier和Render的抽象，实际看进去，比较复杂的恰恰是RendererAdapter对两者的调度（不知道设计者实现这些细节花了多久，个人感觉应该需要认真思考一些时间，否则很多细节很难考虑周全的）。

首先来看init{}中，对Adapter的初始化：

```kotlin
class RendererAdapter(builder: Builder) : RecyclerView.Adapter<RecyclerView.ViewHolder>() {

    private val repositoryCount: Int
    internal val repositories: List<Repo<Any>>

    private var data: List<ViewData> = emptyList()
        set(value) {
            field = value
            endPositions = value.getEndsPonints()
        }
    private var endPositions: IntArray = intArrayOf()

    init {
        repositories = builder.repositories
        repositoryCount = repositories.size

        data = getCurrentViewData()
    }

    fun getCurrentViewData(): List<ViewData> =
        repositories.map {
            it.renderer.getData(it.supplier())
        }
}
```

在Builder中，Supplier和Renderer，被放入了一个Repo的对象中，它的作用只是作为一个容器承载Supplier和Renderer，在初始化时，repositories作为所有repo的容器，repositoryCount则记录**有几种类型的Item**。此外，这几种Item类型对应所有的数据（参考LayoutRenderer，数据被额外包装成了ViewData的子类）都被放入了data中。

当data属性被赋值后，endPositions属性也会随之更新，getEndsPonints（）是List的拓展函数：

```Kotlin
fun <VD: ViewData> List<VD>.getEndsPonints(): IntArray =
        getEndsPonints { it.count }

fun <T> List<T>.getEndsPonints(getter: (T) -> Int): IntArray {
    val ends = IntArray(size)
    var lastEndPosition = 0
    for ((i, vd) in withIndex()) {
        lastEndPosition += getter(vd)
        ends[i] = lastEndPosition
    }

    return ends
}
```

endPositions实际上是对不同类型Item各自最后一条数据，在总的List中的position值记录的一个数组。

举例来说，如果列表中有三种数据，（数据类型-数据数量）分别对应（A-3）,(B-4)，（C-5），那么这个size为12的endPositions数组值应该是[2,6,11]。

这种行为最初看来难以理解，它的作用是用来通过postion，作为索引检索其他数据。

此外，RendererAdapter还有额外的2个属性：

```Kotlin
/**
 * Because the order of function [.getItemViewType] and
 * [.onCreateViewHolder] calls is uncertain,
 * only the last Position will be saved for same type.
 */
private val typeToPositionMap = SparseIntArray()
private val presenterForViewHolder: MutableMap<RecyclerView.ViewHolder, Repo<Any>> = HashMap()
```

这两个属性和上面的endPositions这三个哥们，在最初来看我不了解它们的意义，往下看，它们存在的意义就是——检索。

### getItemViewType：如何获取Layout类型

来看看RendererAdapter中对ItemViewType的处理逻辑，以及其内部用到的resolveIndices函数：

```kotlin
override fun getItemViewType(position: Int): Int {
    val (resolvedRepositoryIndex, resolvedItemIndex) = resolveIndices(position, endPositions)
    val type = repositories[resolvedRepositoryIndex].renderer.getItemViewType(
            data[resolvedRepositoryIndex], resolvedItemIndex)
    typeToPositionMap.put(type, position)
    return type
}

// 相对比较复杂，简单了解即可
fun resolveIndices(position: Int, endPos: IntArray): Pair<Int, Int> {
    if (position < 0 || position >= endPos.last()) {
        throw IndexOutOfBoundsException(
                "Asked for position $position while count is ${endPos.last()}")
    }

    var arrayIndex = Arrays.binarySearch(endPos, position)
    if (arrayIndex >= 0) {
        do {
            arrayIndex++
        } while (endPos[arrayIndex] == position)
    } else {
        arrayIndex = arrayIndex.inv()
    }

    val resolvedRepositoryIndex = arrayIndex
    val resolvedItemIndex = if (arrayIndex == 0) position else position - endPos[arrayIndex - 1]

    return resolvedRepositoryIndex to resolvedItemIndex
}
```

getItemViewType需要知道当前Position的Item所对应Layout的类型，以上文的（A-3）,(B-4)，（C-5）为例，如果position为2，那么，   val (resolvedRepositoryIndex, resolvedItemIndex)  = Pair(0,2),接下来，就能获取到对应的A数据对应的Renderer，并且通过A数据对应的Renderer获取对应的type。

### onCreateViewHolder和onBindViewHolder

```kotlin
override fun onCreateViewHolder(parent: ViewGroup,
                                viewType: Int): RecyclerView.ViewHolder {
    val typeLastPosition = typeToPositionMap.get(viewType)

    val (resolvedRepositoryIndex, _) = resolveIndices(typeLastPosition, endPositions)

    return repositories[resolvedRepositoryIndex].renderer.onCreateViewHolder(parent, viewType)
}

override fun onBindViewHolder(holder: RecyclerView.ViewHolder, position: Int) {
    val (resolvedRepositoryIndex, resolvedItemIndex) = resolveIndices(position, endPositions)
    val repo = repositories[resolvedRepositoryIndex]
    presenterForViewHolder[holder] = repo
    repo.renderer.bind(data[resolvedRepositoryIndex], resolvedItemIndex, holder)
}
```

正如上面的getItemViewType，通过三剑客，我们能检索到position对应Item的Renderer，然后，我们只需要把Adapter需要的行为委托给不同的Renderer处理即可。

到这里，基本的思路已经捋的差不多了，关于DataBinding的支持以及数据的更新，整理好思路，再看对应位置的代码，应该也不难理解。

* * *
## 小结

在经历过思考后，我很钦佩 [Yumenokanata](https://github.com/Yumenokanata) 对于 [DslAdapter](https://github.com/Yumenokanata/DslAdapter) 的设计。学习过程中，我受益匪浅，它的优点在我看来有以下几点（重要程度从低到高）：

* 1.纯Kotlin编写，DSL的Adapter的实现方式；
* 2.对Kotlin的使用，一些细节之处的代码规范值得学习；
* 3.Kotlin代码中对函数式思想的应用；
* 4.设计的思想，对复杂需求的抽象。

上述列表中，在clone代码的最初，我的目标是1和4，所幸我在学习的还同时收获了2，3（对于Kotlin的这些特性，算是查漏补缺把），我在此总结自己的所得，希望对于看到这里的各位能有所帮助。


