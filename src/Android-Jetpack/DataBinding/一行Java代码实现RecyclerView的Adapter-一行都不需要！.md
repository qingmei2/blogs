> 本篇文章已授权微信公众号 guolin_blog （郭霖）独家发布
## 前言

**听说这种【一行代码实现xxx】用烂的标题总是能够吸引到更多的关注。**

在批判笔者这种行为之前，我们先来总结一下目前Android开发中通过RecyclerView列表的几种常见实现方式。

* 1.直接使用原生RecyclerView提供的API，自己实现RecyclerView的Adapter和ViewHolder。
* 2.使用网上比较火的三方库，类似一行代码实现上拉加载更多，下拉刷新，xxx,xxx的RecyclerViewAdapter；或者个人开发者基于此类，再度封装的BaseAdapter。
* 3.使用Databinding，写一个一劳永逸的Adapter，从此告别Adapter的多次实现。

笔者阐述一下个人对于上述3种列表的实现方式的评价:

> 1.直接使用原生RecyclerView提供的API，自己实现RecyclerView的Adapter和ViewHolder。

简单而又直接，无论是列表的实现者，还是后来代码的维护者，都能第一时间理解代码的意图，但是弊端很明显，那就是Adapter类和ViewHolder类过于繁多，每一个列表都需要一个对应的Adapter和ViewHolder。

对于偷懒的程序员来讲，这种重复性的行为显然是难以令人接受的。

> 2.使用网上比较火的三方库，类似一行代码实现上拉加载更多，下拉刷新，xxx,xxx的RecyclerViewAdapter；或者个人开发者基于此类，再度封装的BaseAdapter。

事实上，现在网络上越来越多出现别人封装好的RecyclerViewAdapter或其他工具，恕笔者直言，大多数都略有哗众取宠之嫌，很多都不可避免出现了 **过度封装** 的情况：它也许能够涵括大多数的需求，但是这也恰恰是它致命的弊端，在涉及一些新的功能时，它也许会突然无能为力——过度的封装带来了严重的耦合，这种问题是架构级的。

一个良好的设计需要更多的思考和尝试，更重要的也许是灵活，高度的可拓展性。

在这里笔者推荐一个已经使用了很久的库：[drakeet大神](https://github.com/drakeet) 的 [【MultiType】:An Android library to create multiple item types list views easily and flexibly](https://github.com/drakeet/MultiType)

> 3.使用Databinding，写一个一劳永逸的Adapter，从此告别Adapter的多次实现

DataBinding，google推出的大名鼎鼎的库，也是Android开发中MVVM架构中不可或缺的基础组件之一，它的定义很纯粹，那就是

>  **数据驱动视图**

很遗憾的是，因为MVP模式的便利和简单（是**简单**而不是**简洁**，事实上，MVP开发模式的强大也是掣肘它的原因之一，一个单纯的界面至少需要Contract+MVP多个文件进行配置，还不算dagger2Component+Module文件的配置，随着开发时间的延长，这种疑问逐渐在我脑海中浮现），MVP的拥护者确实比MVVM多一些，DataBinding并未大面积在Android开发中拓展开来，也是因此，笔者也很少看到关于DataBinding的实践和总结的文章。

通过DataBinding实现列表，这种需求的实现已经不是什么难事，网上一搜文章一大堆，但是仍然略微有些复杂。

笔者今天尝试将个人开发过程中，对通过DataBinding实现RecyclerView列表的方式所得进行一次简单的分享，不足之处，欢迎拍砖。

>**注意，本文需要读者有一定的DataBinding的使用基础。**

## DataBinding简介
> 如果您对于DataBinding并不是很熟悉，不用担心，接下来我将尽量用简洁的语言对DataBinding进行简单的介绍，让客官最快了解DataBinding的基本语法和其优势。
如果您已经对DataBinding有过一定的学习，可跳过本小节，直接阅读[正文]部分。

#### 1.DataBinding是什么

Data Binding Library （数据绑定库），旨在减少绑定应用程序逻辑和布局所需的一些耦合性代码。
DataBinding最低支持Android 2.1 (API Level 7)。

#### 2.添加DataBinding的依赖

在build.gradle添加如下声明，添加DataBinding的依赖：
```groovy
android {
     ....
    dataBinding {
        enabled = true
    }
}
```
#### 3.在你的布局文件中绑定数据

我们将activity_main.xml的布局文件进行这样的配置：

```xml
<?xml version="1.0" encoding="utf-8"?>
<layout xmlns:android="http://schemas.android.com/apk/res/android">
   <data>
       <variable name="user" type="com.example.User"/>  //绑定一个User类型的数据
   </data>
   <LinearLayout
       android:orientation="vertical"
       android:layout_width="match_parent"
       android:layout_height="match_parent">
       <TextView android:layout_width="wrap_content"
           android:layout_height="wrap_content"
           android:text="@{user.firstName}"/>  //TextView，显示的内容为user.firstName
       <TextView android:layout_width="wrap_content"
           android:layout_height="wrap_content"
           android:text="@{user.lastName}"/>   //TextView，显示的内容为user.lastName
   </LinearLayout>
</layout>
```
以上文demo为例，在<data>标签中 <variable>描述了一个变量user，它对应的类型为”com.example.User” 
布局中使用“@ { }”语法来书写表达式。如为TextView的文本设置成user. firstName。

> 这看起来就好像是，我们将一个user对象，传给了xml布局文件，布局文件中的控件根据这个对象的对应属性，显示对应的数据。

顺便提供一下User类：
```
public class User {
   public String firstName;
   public String lastName;
   public User(String firstName, String lastName) {
       this.firstName = firstName;
       this.lastName = lastName;
   }
}
```


#### 4.如何绑定数据
从3的代码中，衍生出一个问题，我们如何将user对象传给activity_main的布局文件呢？

我们来看代码，我们在MainActivity中进行这样的配置，以代替Activity传统的setContentView(layoutRes)方法：

```java
@Override
protected void onCreate(Bundle savedInstanceState) {
   super.onCreate(savedInstanceState);
   //MainActivity绑定activity_main布局，类似setContentView()方法
   MainActivityBinding binding = DataBindingUtil.setContentView(this, R.layout.main_activity);
  //初始化User并赋值给Binding(你也可以先暂时理解为赋值给xml布局文件)
   User user = new User("Test", "User");
   binding.setUser(user);
}
```
好了，配置到这里，我们就可以运行demo，然后观察到，MainActivity界面的两个TextView,都成功显示了User的对应属性了！

这种通过数据绑定，从而控制控件显示的方式，相比传统的TextView.setText(user.name)，好处在哪呢，最直观的好处是：

 - 避免Activity中多次使用findViewById，损害了应用性能且令人厌烦。
 - 传统的方式，更新UI数据需切换至UI线程，并将数据分解映射到各个view(比如TextView.setText())，而DataBinding避免了这种麻烦的行为。

#### 5.DataBinding的更多支持

除了上文的说到功能，DataBinding还提供了更多优秀的特性支持，对此请参考官方文档说明（本小节的示例代码也是节选自官方文档），篇幅所限，无法一一举例，还望谅解。

https://developer.android.google.cn/topic/libraries/data-binding/index.html

> 看到这里，虽然你可能还是对DataBinding一头雾水——仅仅是看懂了最基本的使用方式，而没有理解到DataBinding绝对的优势在哪里。

> 没有关系，来看一看，笔者在目前项目中，通过DataBinding对RecyclerView的一种“新的”实现方式，相信有了刚才的趁热打铁，即使您没有使用过DataBinding,对接下来的代码也能够大概看明白。

这之后，如果您对DataBinding感兴趣，再尝试花时间去慢慢研究和使用它。

# 正文
### 如何一行Java代码都不要，实现RecyclerView列表呢？

> 真的不需要Java代码就能实现列表吗？

对不起朋友们，我骗了你们，是需要代码的。

## 打死标题党！

请在您挥舞下手中40米长的大刀之前，来看一下笔者实现列表的代码：

## 单类型列表的实现

先看下MainActivity的java代码

```Java
public class MainActivity extends AppCompatActivity {
    
   //要展示的数据源
    public final ObservableArrayList<Student> showDatas = new ObservableArrayList<>();
   
    {
        //初始化数据源
        for (int i = 0; i < 20; i++) {
            students.add(new Student("学生:" + i));
        }
        showDatas.addAll(students);
    }

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        //完成数据和布局的绑定
        ActivityMainBinding binding = DataBindingUtil.setContentView(this, R.layout.activity_main);
        binding.setActivity(this);
    }

    public void onBindItem(ViewDataBinding binding, Object data, int position) {
        binding.getRoot().setOnClickListener(v -> Toast.makeText(this, data.toString(), Toast.LENGTH_SHORT).show());
    }

    //数据的实体类
    public class Student {
          public String name;
          public Student(String name) {
            this.name = name;
          }
      }

}
```

笔者保证，除了MainActivity.java类外，不再有任何MainActivity相关的Java文件（比如MainPresenter ，MainModel ， MainActivityListAdapter等）。

运行App,让我们来看一下MainActivity的UI:

![SingleType.png](https://upload-images.jianshu.io/upload_images/7293029-603b368a243cf449.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

现在我们点击单个Item，Item还会响应对应的点击事件——弹出一个toast,并打印对应的Student对象。

熟悉DataBinding的朋友们肯定有一些猜测了，我们来看一下对应的activity_main.xml文件：

```xml
<?xml version="1.0" encoding="utf-8"?>
<layout xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:app="http://schemas.android.com/apk/res-auto"
    xmlns:tools="http://schemas.android.com/tools">

    <data>
        <variable
            name="activity"
            type="com.qingmei2.simplerecyclerview.MainActivity" />
    </data>

    <android.support.v7.widget.RecyclerView
        android:layout_width="match_parent"
        android:layout_height="match_parent"
        app:items="@{activity.showDatas}"
        app:layoutManager="@string/linear_layout_manager"
        app:itemLayout="@{@layout/item_student_list}"
        app:onBindItem="@{activity::onBindItem}" />
</layout>
```
以及对应的item的layout.xml：
```
<?xml version="1.0" encoding="utf-8"?>
<layout xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:app="http://schemas.android.com/apk/res-auto"
    xmlns:tools="http://schemas.android.com/tools">

    <data>
        <variable
            name="data"
            type="com.qingmei2.simplerecyclerview.Student" />
    </data>

    <LinearLayout
        android:layout_width="match_parent"
        android:layout_height="40dp"
        android:orientation="vertical">
        
         <!--显示人名的TextView-->
         <TextView
            android:layout_width="match_parent"
            android:layout_height="0dp"
            android:layout_weight="1"
            android:gravity="center_vertical"
            android:padding="8dp"
            android:text="@{data.name}"
            tools:text="小明" />

        <!--Item下方灰色的分割线-->
        <View
            android:layout_width="match_parent"
            android:layout_height="0.5dp"
            android:background="#ccc" />

    </LinearLayout>
</layout>
```
不可否认的是，作为MainActivity的一个列表，笔者确实没有使用Java代码实现RecyclerView的Adapter和ViewHolder，以及设置LayoutManager，哪怕一行都没有。

我们先来看一下魔法的根源，即activity_main.xml文件的RecyclerView的配置，一切的实现都来源于下面的四条属性：

```
    <android.support.v7.widget.RecyclerView
        android:layout_width="match_parent"
        android:layout_height="match_parent"
        app:items="@{activity.showDatas}"  //要显示的数据源
        app:layoutManager="@string/linear_layout_manager"  //指定LayoutManager
        app:itemLayout="@{@layout/item_student_list}"  //数据展示在哪个布局上
        app:onBindItem="@{activity::onBindItem}" />  //更多配置，比如我想设置点击事件，或者引用Context
```
我们抛开怎么实现，先阐述这四条属性，为何就能展示一个完整的列表呢？

>  //1.要显示的数据源
app:items="@{activity.showDatas}" 

我们从MainActivity中可以看到，activity.showDatas实际上就是ObservableArrayList<Student>类型的List, ObservableArrayList本身就是ArrayList的子类，这个属性的意义在于，告诉RecyclerView：

**你需要展示的列表所需要的数据都在这里了,这个List有多少条数据，你就展示多少个item。**

显然，我们在代码中，通过模拟网络请求的结果，给list初始化了20条Student数据,因此，RecyclerView知道，需要展示20条数据，并为其创建20条item展示出来。

那么数据有了，问题来了，数据如何展示给用户呢？

因此我们需要配置item对应的layout文件：

> //2.数据展示在哪个布局上
app:itemLayout="@{@layout/item_student_list}"

我们将item_student_list.xml——item的布局文件传给了RecyclerView,RecyclerView就知道了如何将数据展示在item上。

那么，数据如何展示在item上的呢？请往上翻，我们可以看到，item的layout文件中，也已经将我们要展示的Student作为data传进了item的LayoutBinding中，layout的子控件就会知道，该如何展示student的数据了。比如，将student的name展示在TextView上：

```
         <!--显示人名的TextView-->
         <TextView
            android:layout_width="match_parent"
            android:layout_height="0dp"
            android:layout_weight="1"
            android:gravity="center_vertical"
            android:padding="8dp"
            android:text="@{data.name}"
            tools:text="小明" />
```
现在数据和布局都有了，RecyclerView还需要知道，如何布局？是LinearLayout还是GridLayout呢？

很简单，我们传进来就可以了：

>  //3.指定LayoutManager
app:layoutManager="@string/linear_layout_manager" 

简单明了，我们指定使用了LinearLayoutManager

其实按理说，上述3条属性已经够用了，但是我们还需要考虑到一些拓展的需求，比如点击事件，或者和Activity/Fragment的联动?

> //4 更多配置的回调
app:onBindItem="@{activity::onBindItem}" 

我们声明了一个回调，并在MainActivity中实现了这个回调：
```
 public void onBindItem(ViewDataBinding binding, Object data, int position) {
        binding.getRoot().setOnClickListener(v -> Toast.makeText(this, data.toString(), Toast.LENGTH_SHORT).show());
    }
```
demo中很简单，我们只声明了一个点击事件。事实上，也许有更多的需求，比如根据item中控件状态的变更（比如checkbox等），来做出对应的行为，我们在回调中声明了3个参数：

* ViewDataBinding binding：item的Binding，通过向下转型即可获得对应的Binding对象，比如本文的ItemStudentListBinding
* Object data : item对应的数据，通过向下转型即可获得对应的对象，比如本文中可以转换为Student
* int position：很明显，就是item在list中的索引

示例代码:
```
    public void onBindItem(ViewDataBinding binding, Object data, int position) {
        ItemStudentListBinding bind = (ItemStudentListBinding) binding;
        Student student = (Student) data;
        //点击item，toast，打印学生的name
        bind .getRoot().setOnClickListener(v -> Toast.makeText(this, student.name, Toast.LENGTH_SHORT).show());
    }
```
看起来很像RecyclerView的Adapter的onBindViewHolder方法？原理也确实如此，只不过是将这个接口暴漏出来，方便开发者进行特殊处理。

相信，这四个属性的提供，足以实现各种各样 **单类型列表** 的需求了。

## 成功男人背后的女人

如果您的项目中使用了DataBinding，从此之后，您项目中的RecyclerView都将是这么的简洁：

```
<android.support.v7.widget.RecyclerView
        android:layout_width="match_parent"
        android:layout_height="match_parent"
        app:items="@{activity.showDatas}"
        app:layoutManager="@string/linear_layout_manager"
        app:itemLayout="@{@layout/item_student_list}"
        app:onBindItem="@{activity::onBindItem}" />
```
在此之前，您需要进行一些配置，这些配置我已经连同本文的Demo一起放在了我的github上，供您参考：

[本文的Demo源码：MultiTypeBindings](https://github.com/qingmei2/MultiTypeBindings)

请先将目光转回到本文中，我们一起实现几个简单的类：

### 1.添加MultiType的依赖

正如前言所说， [MultiType](https://github.com/drakeet/MultiType)是一个灵活且可以高度拓展的库，本文的demo也是基于其进行的开发：

```groovy
    implementation 'me.drakeet.multitype:multitype:3.3.0'
    implementation 'com.annimon:stream:1.1.9'
```
同时，为了代码简洁，我添加了Java8的StreamAPI的向下兼容库的依赖，您也可以选择不添加，只需要将对应的Java8方法转换为普通的方法即可，而不会影响对应的功能。

当然，我们不要忘记在android的目录下添加databinding的支持:

```groovy
android {
    dataBinding {
        enabled = true
    }
}
```

### 2.实现DataBindingItemViewBinder和DataBindingViewHolder

```Java
public class DataBindingItemViewBinder<T, DB extends ViewDataBinding>
        extends ItemViewBinder<T, DataBindingViewHolder<DB>> {

    private final Delegate<T, DB> delegate;

    public DataBindingItemViewBinder(Delegate<T, DB> delegate) {
        this.delegate = delegate;
    }

    public DataBindingItemViewBinder(BiFunction<LayoutInflater, ViewGroup, DB> factory,
                                     OnBindItem<T, DB> binder) {
        this(new SimpleDelegate<>(factory, binder));
    }

    public DataBindingItemViewBinder(@LayoutRes int resId, OnBindItem<T, DB> binder) {
        this((inflater, parent) -> DataBindingUtil.inflate(inflater, resId, parent, false), binder);
    }

    @NonNull
    @Override
    protected DataBindingViewHolder<DB> onCreateViewHolder(@NonNull LayoutInflater inflater,
                                                           @NonNull ViewGroup parent) {
        return new DataBindingViewHolder<>(delegate.onCreateDataBinding(inflater, parent));
    }

    @Override
    protected void onBindViewHolder(@NonNull DataBindingViewHolder<DB> holder, @NonNull T item) {
        final DB binding = holder.dataBinding;
        binding.setVariable(BR.data, item);//数据绑定对应的item layout
        delegate.onBind(binding, item, holder.getAdapterPosition());//回调
        binding.executePendingBindings();
    }

    public interface Delegate<T, DB extends ViewDataBinding> {
        DB onCreateDataBinding(@NonNull LayoutInflater inflater, @NonNull ViewGroup parent);

        void onBind(@NonNull DB dataBinding, @NonNull T item, int position);
    }

    public interface OnBindItem<T, DB extends ViewDataBinding> {
        void bind(DB dataBinding, T data, int position);
    }

    private static class SimpleDelegate<T, DB extends ViewDataBinding> implements Delegate<T, DB> {
        private final BiFunction<LayoutInflater, ViewGroup, DB> factory;
        private final OnBindItem<T, DB> binder;

        SimpleDelegate(BiFunction<LayoutInflater, ViewGroup, DB> factory, OnBindItem<T, DB> binder) {
            this.factory = factory;
            this.binder = binder;
        }

        @Override
        public DB onCreateDataBinding(@NonNull LayoutInflater inflater, @NonNull ViewGroup parent) {
            return factory.apply(inflater, parent);
        }

        @Override
        public void onBind(@NonNull DB dataBinding, @NonNull T item, int position) {
            binder.bind(dataBinding, item, position);
        }
    }
}
```

```
public class DataBindingViewHolder<T extends ViewDataBinding> extends RecyclerView.ViewHolder {
    public final T dataBinding;

    public DataBindingViewHolder(T binding) {
        super(binding.getRoot());

        dataBinding = binding;
    }
}
```
这两个类的作用就是**通过代理的方式实现了通用的Adapter和ViewHolder**,我们实现了它们，只要不是过于复杂的列表，我们都**不再需要实现RecyclerView的Adapter和ViewHolder**了。

我将不会对这两个核心类有过多的讲解，因为它们对于熟悉Databinding的使用者来说，并不难以理解。

如果您对于DataBinding并不是很熟悉，笔者建议您暂时先新建这两个类，并将代码复制上去——当您能够驾轻就熟地使用这个工具后，再尝试研究它的原理，相信我，它的原理本身也并不复杂。

### 3.实现对应的BindingAdapter和Linker类
```
public class RecyclerViewBindingAdapter {

    public static class BindableVariables extends BaseObservable {
        @Bindable
        public Object data;
    }

    @BindingAdapter({"itemLayout", "onBindItem"})
    public static void setAdapter(RecyclerView view, int resId, DataBindingItemViewBinder.OnBindItem onBindItem) {
        final MultiTypeAdapter adapter = getOrCreateAdapter(view);
        //noinspection unchecked
        adapter.register(Object.class, new DataBindingItemViewBinder(resId, onBindItem));
    }

    private static MultiTypeAdapter getOrCreateAdapter(RecyclerView view) {
        if (view.getAdapter() instanceof MultiTypeAdapter) {
            return (MultiTypeAdapter) view.getAdapter();
        } else {
            final MultiTypeAdapter adapter = new MultiTypeAdapter();
            view.setAdapter(adapter);
            return adapter;
        }
    }

    @BindingAdapter({"linkers", "onBindItem"})
    public static void setAdapter(RecyclerView view, List<Linker> linkers, DataBindingItemViewBinder.OnBindItem onBindItem) {
        final MultiTypeAdapter adapter = getOrCreateAdapter(view);
        //noinspection unchecked
        final ItemViewBinder[] binders = Stream.of(linkers)
                .map(Linker::getLayoutId)
                .map(v -> new DataBindingItemViewBinder(v, onBindItem))
                .toArray(ItemViewBinder[]::new);
        //noinspection unchecked
        adapter.register(Object.class)
                .to(binders)
                .withLinker(o -> Stream.of(linkers)
                        .map(Linker::getMatcher)
                        .indexed()
                        .filter(v -> v.getSecond().apply(o))
                        .findFirst()
                        .map(IntPair::getFirst)
                        .orElse(0));
    }

    @BindingAdapter("items")
    public static void setItems(RecyclerView view, List items) {
        final MultiTypeAdapter adapter = getOrCreateAdapter(view);
        adapter.setItems(items == null ? Collections.emptyList() : items);
        adapter.notifyDataSetChanged();
    }
}

```

```
public class Linker {
    private final Function<Object, Boolean> matcher;
    private final int layoutId;

    public static Linker of(Function<Object, Boolean> matcher, int layoutId) {
        return new Linker(matcher, layoutId);
    }

    public Linker(Function<Object, Boolean> matcher, int layoutId) {
        this.matcher = matcher;
        this.layoutId = layoutId;
    }

    public Function<Object, Boolean> getMatcher() {
        return matcher;
    }

    public int getLayoutId() {
        return layoutId;
    }
}
```
DataBinding提供了@BindingAdapter注解，用于**绑定View**和**拓展对应的行为**，关于这个注解，我们通过百度或者谷歌，都能搜到大量的学习资料，在此也不多赘述。

我们可以看到，RecyclerViewBindingAdapter 这个类中，声明了我们刚刚认识并了解了的几个属性，比如“itemLayout”，“onBindItem”，“items”等属性，声明了这些属性的静态方法，作用也就是自动创建对应的Adapter，然后进行数据与视图的绑定。

我们可以看到除了几个熟悉的属性，我们还声明了"linkers"属性，以及声明了一个Linker类，它们的作用是用来实现**多类型列表**，我们下文将会提到。

### 4.其他的配置

1 2 3 的步骤已经将核心的类都声明完毕，接下来我们需要在attr.xml文件中声明我们需要用到的属性：

```xml
<?xml version="1.0" encoding="utf-8"?>
<resources>

    <declare-styleable name="RecyclerView">
        <attr name="items" format="reference" />
        <attr name="itemLayout" format="reference" />
        <attr name="linkers" format="reference" />
        <attr name="layoutManager" format="reference" />
        <attr name="onBindItem" format="reference" />
    </declare-styleable>

</resources>
```

这样，我们在xml文件中，直接通过代码提示的功能，为RecyclerView赋予对应的配置了。

然后，在values.xml文件中，声明好我们要引用的layoutManager:

```xml
<?xml version="1.0" encoding="utf-8"?>
<resources>
    <string name="linear_layout_manager">android.support.v7.widget.LinearLayoutManager</string>
    <string name="grid_layout_manager">android.support.v7.widget.GridLayoutManager</string>
</resources>
```

配置到这里，上面demo中我们实现的功能就已经可以实现了，我们的这些配置类，都是一次声明，之后项目中无需再进行处理的，也就是说，随着项目中列表越来越多，我们将会节省越来越多的代码。

最后，不管再多的RecyclerView，我们都只需要配置好xml文件中RecyclerView对应的四条属性，然后，告别繁多的Adapter，LayoutManager和ViewHolder，and enjoy coding!

(PS，对于Activity的onBindItem的回调方法，复杂的需求也许会导致很臃肿，比如状态的判断处理，这也是一直在思考能否再简化的地方，有思路的朋友望请不吝赐教！)

## 多类型列表需要几行代码？

> 大概，也是0行吧。

一个简单的demo：

![multitype.png](https://upload-images.jianshu.io/upload_images/7293029-659211901805e6cf.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

这仍然是一个RecyclerView列表，不同的是，它需要展示Teacher和Student两种数据（因为笔者懒，所以2种数据没有打乱排列，但是请相信，他们仍处于同一个RecyclerView，并对应不同的布局和逻辑处理）。

让我们看一看代码：

```
public class MainActivity extends AppCompatActivity {
    //要展示数据源
    public final ObservableArrayList<Object> showDatas = new ObservableArrayList<>();
    //Linker对象的list，用来管理item展示的逻辑
    public final ObservableArrayList<Linker> linkers = new ObservableArrayList<>();

    public final List<Student> students = new ArrayList<>();
    public final List<Teacher> teachers = new ArrayList<>();


    {
        for (int i = 0; i < 20; i++) {
            students.add(new Student("学生:" + i));
        }
        for (int j = 0; j < 5; j++) {
            teachers.add(new Teacher("教师:" + j, "年龄：" + (20 + j)));
        }
        linkers.add(
                new Linker(
                        o -> o instanceof Student,
                        R.layout.item_student_list
                )//如果item的数据是Student类型，使用item_student_list布局
        );
        linkers.add(
                new Linker(    
                        o -> o instanceof Teacher,
                        R.layout.item_teacher_list
                )//如果item的数据是Teacher类型，使用item_teacher_list布局
        );
        showDatas.addAll(students);
        showDatas.addAll(teachers);
    }

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        ActivityMainBinding binding = DataBindingUtil.setContentView(this, R.layout.activity_main);
        binding.setActivity(this);
    }

    public void onBindItem(ViewDataBinding binding, Object data, int position) {
        binding.getRoot().setOnClickListener(v -> Toast.makeText(MainActivity.this, data.toString(), Toast.LENGTH_SHORT).show());
    }
}
```

我们看到，依然没有Adapter，ViewHolder（如果是常规实现方式，这里应该是2种ViewHolder的类），以及LayoutManager。

看一下布局文件：

```
        <android.support.v7.widget.RecyclerView
            android:layout_width="match_parent"
            android:layout_height="match_parent"
            app:items="@{activity.showDatas}"
            app:layoutManager="@string/linear_layout_manager"
            app:linkers="@{activity.linkers}"  //请注意这行
            app:onBindItem="@{activity::onBindItem}" />
```

和单类型列表相比，我们少了

>app:itemLayout="@{@layout/item_student_list}"

多了

> app:linkers="@{activity.linkers}" 

很好理解，对于多类型列表的展示，我们会定义多个不同item的layout布局文件，因此我们不能单纯的为RecyclerView赋予固定的布局，而是赋予其不同item的所有layout文件

> R.layout.item_student_list    
> R.layout.item_teacher_list

接下来需要思考的问题是，我们如何得知每一个item需要使用哪种类型的布局呢?

我们可以通过一个函数，来判断item数据的类型，如果是Student类型，就使用R.layout.item_student_list ，如果是Teacher类型，就使用R.layout.item_teacher_list。

因此我们衍生出了Linker类（见上文），它包含了一个LayoutRes属性和一个Function<Object, Boolean>函数，我们初始化时，根据数据对应的类型进行判断，如果函数的返回值为true，就使用其内部的LayoutRes并进行展示：

```Java
linkers.add(new Linker(
                        o -> o instanceof Student,
                        R.layout.item_student_list
                )//如果item的数据是Student类型，使用item_student_list布局
);
linkers.add(new Linker(    
                        o -> o instanceof Teacher,
                        R.layout.item_teacher_list
                )//如果item的数据是Teacher类型，使用item_teacher_list布局
);
```

## 小结

在使用MVVM模式进行项目开发的大半年里，收获良多，在此尤其感谢同事Z0君对自己的很多指点（事实上，本文的实现完全是来源于TA的思路，笔者只不过照搬，理解和阐述分享而已），同时感谢项目中共同一起开发的小伙伴们，共勉。

在本文的标题选择上，笔者选择了这么强目的性的标题，也确实希望能够有更多朋友能够一起打开本文，阅读并共同探讨，希望以文章的内容能够表达笔者对此的歉意。

真诚地感谢，您能够坚持阅读到这里，对文章内容的肯定，就是对作者最大的鼓励。

本文Demo的源码传送门，有兴趣的朋友可以拉下来运行一下，希望能够提供一定的思路：

https://github.com/qingmei2/MultiTypeBindings

