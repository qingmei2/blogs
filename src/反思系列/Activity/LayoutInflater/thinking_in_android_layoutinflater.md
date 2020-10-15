# 反思|Android LayoutInflater机制的设计与实现

> **反思** 系列博客是我的一种新学习方式的尝试，该系列起源和目录请参考 [这里](https://github.com/qingmei2/android-programming-profile/blob/master/src/%E5%8F%8D%E6%80%9D%E7%B3%BB%E5%88%97/%E5%8F%8D%E6%80%9D%7C%E7%B3%BB%E5%88%97%E7%9B%AE%E5%BD%95.md) 。

## 概述

`Android`体系本身非常宏大，源码中值得思考和借鉴之处众多。以`LayoutInflater`本身为例，其整个流程中除了调用`inflate()`函数 **填充布局** 功能之外，还涉及到了 **应用启动**、**调用系统服务**（进程间通信）、**对应组件作用域内单例管理**、**额外功能扩展** 等等一系列复杂的逻辑。

本文笔者将针对`LayoutInlater`的整个设计思路进行描述，其整体结构如下图：

![](https://raw.githubusercontent.com/qingmei2/qingmei2-blogs-art/master/android/core/layoutInflater/image.06s2w8rrdv4t.png)

## 整体思路

### 1、创建流程

顾名思义，`LayoutInflater`的作用就是 **布局填充器** ，其行为本质是调用了`Android`本身提供的 **系统服务**。而在`Android`系统的设计中，获取系统服务的实现方式就是通过`ServiceManager`来取得和对应服务交互的`IBinder`对象，然后创建对应系统服务的代理。

`Android`应用层将系统服务注册相关的`API`放在了`SystemServiceRegistry`类中，而将注册服务行为的代码放在了`ContextImpl`类中，`ContextImpl`类实现了`Context`类下的所有抽象方法。

`Android`应用层还定义了一个`Context`的另外一个子类：`ContextWrapper`，`Activity`、`Service`等组件继承了`ContextWrapper`, 每个`ContextWrapper`的实例有且仅对应一个`ContextImpl`，形成一一对应的关系，该类是 **装饰器模式** 的体现：保证了`Context`类公共功能代码和不同功能代码的隔离。

此外，虽然`ContextImpl`类作为`Context`类公共`API`的实现者，`LayoutInlater`的获取则交给了`ContextThemeWrapper`类，该类中将`LayoutInlater`的获取交给了一个成员变量，保证了单个组件 **作用域内的单例**。

### 2、布局填充流程

开发者希望直接调用`LayoutInflater#inflate()`函数对布局进行填充，该函数作用是对`xml`文件中标签的解析，并根据参数决定是否直接将新创建的`View`配置在指定的`ViewGroup`中。

一般来说，一个`View`的实例化依赖`Context`上下文对象和`attr`的属性集，而设计者正是通过将上下文对象和属性集作为参数，通过 **反射** 注入到`View`的构造器中对`View`进行创建。

除此之外，考虑到 **性能优化** 和 **可扩展性**，设计者为`LayoutInflater`设计了一个`LayoutInflater.Factory2`接口，该接口设计得非常巧妙：在`xml`解析过程中，开发者可以通过配置该接口对`View`的创建过程进行拦截：**通过new的方式创建控件以避免大量地使用反射**，亦或者 **额外配置特殊标签的解析逻辑以创建特殊组件**（比如`Fragment`）。

> `LayoutInflater.Factory2`接口在`Android SDK`中的应用非常普遍，`AppCompatActivity`和`FragmentManager`就是最有力的体现，`LayoutInflater.inflate()`方法的理解虽然重要，但笔者窃以为`LayoutInflater.Factory2`的重要性与其相比不逞多让。

对于`LayoutInflater`整体不甚熟悉的开发者而言，本小节文字描述似乎晦涩难懂，且难免有是否过度设计的疑惑，但这些文字的本质却是布局填充流程整体的设计思想，**读者不应该将本文视为源码分析，而应该将自己代入到设计的过程中** 。

## 创建流程

![](https://raw.githubusercontent.com/qingmei2/qingmei2-blogs-art/master/android/core/layoutInflater/image.u70digob65.png)

### 1.Context：系统服务的提供者

上文提到，`LayoutInflater`作为系统服务之一，获取方式是通过`ServiceManager`来取得和对应服务交互的`IBinder`对象，然后创建对应系统服务的代理。

`Binder`机制相关并非本文的重点，读者可以注意到，`Android`的设计者将获取系统服务的接口交给了`Context`类，意味着开发者可以通过任意一个`Context`的实现类获取系统服务，包括不限于`Activity`、`Service`、`Application`等等：

```java
public abstract class Context {
  // 获取系统服务
  public abstract Object getSystemService(String name);
  // ......
}
```

读者需要理解，`Context`类地职责并非只针对 **系统服务** 进行提供，还包括诸如 **启动其它组件**、**获取SharedPerferences** 等等，其中大部分功能对于`Context`的子类而言都是公共的，因此没有必要每个子类都对其进行实现。

`Android`设计者并没有直接通过继承的方式将公共业务逻辑放入`Base`类供组件调用或者重写，而是借鉴了 **装饰器模式** 的思想：分别定义了`ContextImpl`和`ContextWrapper`两个子类：

![](https://raw.githubusercontent.com/qingmei2/qingmei2-blogs-art/master/android/core/layoutInflater/image.67b4c2y6i1l.png)

### 2.ContextImpl：Context的公共API实现

`Context`的公共`API`的实现都交给了`ContextImpl`，以获取系统服务为例，`Android`应用层将系统服务注册相关的`API`放在了`SystemServiceRegistry`类中，而`ContextImpl`则是`SystemServiceRegistry#getSystemService`的唯一调用者：

```java
class ContextImpl extends Context {
    // 该成员即开发者使用的`Activity`等外部组件
    private Context mOuterContext;

    @Override
    public Object getSystemService(String name) {
        return SystemServiceRegistry.getSystemService(this, name);
    }
}
```

这种设计使得 **系统服务的注册**（`SystemServiceRegistry`类） 和 **系统服务的获取**（`ContextImpl`类） 在代码中只有一处声明和调用，大幅降低了模块之间的耦合。

### 3.ContextWrapper：Context的装饰器

`ContextWrapper`则是`Context`的装饰器，当组件需要获取系统服务时交给`ContextImpl`成员处理，伪代码实现如下：

```java
// class Activity extends ContextWrapper
class ContextWrapper extends Context {
    // 1.将 ContextImpl 作为成员进行存储
    public ContextWrapper(ContextImpl base) {
        mBase = base;
    }

    ContextImpl mBase;

    // 2.系统服务的获取统一交给了ContextImpl
    @Override
    public Object getSystemService(String name) {
      return mBase.getSystemService(name);
    }
}
```

`ContextWrapper`装饰器的初始化如何实现呢？每当一个`ContextWrapper`组件（如`Activity`）被创建时，都为其创建一个对应的`ContextImpl`实例，伪代码实现如下:

```java
public final class ActivityThread {

  // 每当`Activity`被创建
  private Activity performLaunchActivity() {
      // ....
      // 1.实例化 ContextImpl
      ContextImpl appContext = new ContextImpl();
      // 2.将 activity 注入 ContextImpl
      appContext.setOuterContext(activity);
      // 3.将 ContextImpl 也注入到 activity中
      activity.attach(appContext, ....);
      // ....
  }
}
```

> 读者应该注意到了第3步的`activity.attach(appContext, ...)`函数，该函数很重要，在【布局流程】一节中会继续引申。

### 4.组件的局部单例

读者也许注意到，对于单个`Activity`而言，多次调用`activity.getLayoutInflater()`或者`LayoutInflater.from(activity)`，获取到的`LayoutInflater`对象都是单例的——对于涉及到了跨进程通信的系统服务而言，通过作用域内的单例模式保证以节省性能是完全可以理解的。

设计者将对应的代码放在了`ContextWrapper`的子类`ContextThemeWrapper`中，该类用于方便开发者为`Activity`配置自定义的主题，除此之外还通过一个成员持有了一个`LayoutInflater`对象：

```java
// class Activity extends ContextThemeWrapper
public class ContextThemeWrapper extends ContextWrapper {
  private Resources.Theme mTheme;
  private LayoutInflater mInflater;

  @Override
  public Object getSystemService(String name) {
      // 保证 LayoutInflater 的局部单例
      if (LAYOUT_INFLATER_SERVICE.equals(name)) {
          if (mInflater == null) {
              mInflater = LayoutInflater.from(getBaseContext()).cloneInContext(this);
          }
          return mInflater;
      }
      return getBaseContext().getSystemService(name);
  }
}
```

而无论`activity.getLayoutInflater()`还是`LayoutInflater.from(activity)`，其内部最终都执行的是`ContextThemeWrapper#getSystemService`(前者和`PhoneWindow`还有点关系，这个后文会提), 因此获取到的`LayoutInflater`自然是同一个对象了：

```Java
public abstract class LayoutInflater {
  public static LayoutInflater from(Context context) {
      return (LayoutInflater) context.getSystemService(Context.LAYOUT_INFLATER_SERVICE);
  }
}
```

## 布局填充流程

上一节我们提到了`Activity`启动的过程，这个过程中不可避免的要创建一个窗口，最终UI的布局都要展示在这个窗口上，`Android`中通过定义了`PhoneWindow`类对这个UI的窗口进行描述。

### 1.PhoneWindow：setContentView()的真正实现

`Activity`将布局填充相关的逻辑委托给了`PhoneWindow`，`Activity`的`setContentView()`函数，其本质是调用了`PhoneWindow`的`setContentView()`函数。

```java
public class PhoneWindow extends Window {

   public PhoneWindow(Context context) {
       super(context);
       mLayoutInflater = LayoutInflater.from(context);
   }

   // Activity.setContentView 实际上是调用了 PhoneWindow.setContentView()
   @Override
   public void setContentView(int layoutResID) {
       // ...
       mLayoutInflater.inflate(layoutResID, mContentParent);
   }
}
```

读者需要清楚，`activity.getLayoutInflater()`和`activity.setContentView()`等方法都使用到了`PhoneWindow`内部的`LayoutInflater`对象，而`PhoneWindow`内部对`LayoutInflater`的实例化，仍然是调用`context.getSystemService()`方法，因此和上一小节的结论并不冲突：

> 而无论`activity.getLayoutInflater()`还是`LayoutInflater.from(activity)`，其内部最终都执行的是`ContextThemeWrapper#getSystemService`。

`PhoneWindow`是如何实例化的呢，读者认真思考可知，一个`Activity`对应一个`PhoneWindow`的UI窗口，因此当`Activity`被创建时，`PhoneWindow`就被需要被创建了，执行时机就在上文的`ActivityThread.performLaunchActivity()`中：

```java
public final class ActivityThread {

  // 每当`Activity`被创建
  private Activity performLaunchActivity() {
      // ....
      // 3.将 ContextImpl 也注入到 activity中
      activity.attach(appContext, ....);
      // ....
  }
}

public class Activity extends ContextThemeWrapper {

  final void attach(Context context, ...) {
    // ...
    // 初始化 PhoneWindow
    // window构造方法中又通过 Context 实例化了 LayoutInflater
    PhoneWindow mWindow = new PhoneWindow(this, ....);
  }
}
```

设计到这里，读者应该对`LayoutInflater`的整体流程已经有了一个初步的掌握，需要清楚的两点是：

* 1.无论是哪种方式获取到的`LayoutInflater`,都是通过`ContextImpl.getSystemService()`获取的，并且在`Activity`等组件的生命周期内保持单例；  
* 2.即使是`Activity.setContentView()`函数,本质上也还是通过`LayoutInflater.inflate()`函数对布局进行解析和创建。

### 2.inflate()流程的设计和实现

从思想上来看，`LayoutInflater.inflate()`函数内部实现比较简单直观：

```java
public View inflate(@LayoutRes int resource, ViewGroup root, boolean attachToRoot) {
      // ...
}
```

对该函数的参数进行简单归纳如下：第一个参数代表所要加载的布局，第二个参数是`ViewGroup`，这个参数需要与第3个参数配合使用，`attachToRoot`如果为`true`就把布局添加到`ViewGroup`中；若为`false`则只采用`ViewGroup`的`LayoutParams`作为测量的依据却不直接添加到`ViewGroup`中。

从设计的角度上思考，该函数的设计过程中，**为什么需要定义这样的三个参数？为什么这样三个参数就能涵盖我们日常开发过程中布局填充的需求？**

#### 2.1 三个火枪手

对于第一个资源id参数而言，UI的创建必然依赖了布局文件资源的引用，因此这个参数无可厚非。

我们先略过第二个参数，直接思考第三个参数，为什么需要这样一个`boolean`类型的值，以决定是否将创建的`View`直接添加到指定的`ViewGroup`中呢，不设计这个参数是否可以？

换个角度思考，这个问题的本质其实是：是否每个`View`的创建都必须立即添加在`ViewGroup`中？答案当然是否定的，为了保证性能，设计者不可能让所有的`View`被创建后都能够立即被立即添加在`ViewGroup`中，这与目前`Android`中很多组件的设计都有冲突，比如`ViewStub`、`RecyclerView`的条目、`Fragment`等等。

因此，更好的方式应该是可以通过一个`boolean`的开关将整个过程切分成2个小步骤，当`View`生成并根据`ViewGroup`的布局参数生成了对应的测量依据后，开发者可以根据需求手动灵活配置是否立即添加到`ViewGroup`中——这就是第三个参数的由来。

那么`ViewGroup`类型的第二个参数为什么可以为空呢？实际开发过程中，似乎并没有什么场景在填充布局时需要使`ViewGroup`为空？

读者仔细思考可以很容易得出结论，事实上该参数可空是有必要的——对于`Activity`UI的创建而言，根结点最顶层的`ViewGroup`必然是没有父控件的，这时在布局的创建时，就必须通过将`null`作为第二个参数交给`LayoutInlater`的`inflate()`方法，当`View`被创建好后，将`View`的布局参数配置为对应屏幕的宽高：

```java
// DecorView.onResourcesLoaded()函数
void onResourcesLoaded(LayoutInflater inflater, int layoutResource) {
    // ...
    // 创建最顶层的布局时，需要指定父布局为null
    final View root = inflater.inflate(layoutResource, null);
    // 然后将宽高的布局参数都指定为 MATCH_PARENT（屏幕的宽高）
    mDecorCaptionView.addView(root, new ViewGroup.MarginLayoutParams(MATCH_PARENT, MATCH_PARENT));
}
```

现在我们理解了 **为什么三个参数就能涵盖开发过程中布局填充的需求**，接下来继续思考下一个问题，`LayoutInflater`是如何解析`xml`的。

#### 2.2 xml解析流程

`xml`解析过程的思路很简单；

* 1. 首先根据布局文件，生成对应布局的`XmlPullParser`解析器对象；
* 2. 对于单个`View`的解析而言，一个`View`的实例化依赖`Context`上下文对象和`attr`的属性集，而设计者正是通过将上下文对象和属性集作为参数，通过 **反射** 注入到`View`的构造器中对单个`View`进行创建；
* 3. 对于整个`xml`文件的解析而言，整个流程依然通过典型的递归思想，对布局文件中的`xml`文件进行遍历解析，自底至顶对`View`依次进行创建，最终完成了整个`View`树的创建。

单个`View`的实例化实现如下，这里采用伪代码的方式实现：

```java
// LayoutInflater类
public final View createView(String name, String prefix, AttributeSet attrs) {
    // ...
    // 1.根据View的全名称路径，获取View的Class对象
    Class<? extends View> clazz = mContext.getClassLoader().loadClass(name + prefix).asSubclass(View.class);
    // 2.获取对应View的构造器
    Constructor<? extends View> constructor = clazz.getConstructor(mConstructorSignature);
    // 3.根据构造器，通过反射生成对应 View
    args[0] = mContext;
    args[1] = attrs;
    final View view = constructor.newInstance(args);
    return view;
}
```

对于整体解析流程而言，伪代码实现如下：

``` java
void rInflate(XmlPullParser parser, View parent, Context context, AttributeSet attrs) {
  // 1.解析当前控件
  while (parser.next()!= XmlPullParser.END_TAG) {
    final View view = createViewFromTag(parent, name, context, attrs);
    final ViewGroup viewGroup = (ViewGroup) parent;
    final ViewGroup.LayoutParams params = viewGroup.generateLayoutParams(attrs);
    // 2.解析子布局
    rInflateChildren(parser, view, attrs, true);
    // 所有子布局解析结束，将当前控件及布局参数添加到父布局中
    viewGroup.addView(view, params);
  }
}

final void rInflateChildren(XmlPullParser parser, View parent, AttributeSet attrs, boolean finishInflate){
  // 3.子布局作为根布局，通过递归的方式，层级向下一层层解析
  // 继续执行 1
  rInflate(parser, parent, parent.getContext(), attrs, finishInflate);
}
```

至此，一般情况下的布局填充流程到此结束，`inflate()`方法执行完毕，对应的布局文件解析结束，并根据参数配置决定是否直接添加在`ViewGroup`根布局中。

`LayoutInlater`的设计流程到此就结束了吗，当然不是，更精彩更巧妙的设计还尚未登场。

## 拦截机制和解耦策略

### 抛出问题

读者需要清楚的是，到目前为止，我们的设计还遗留了2个明显的缺陷：

* 1.布局的加载流程中，每一个`View`的实例化都依赖了`Java`的反射机制，这意味着额外性能的损耗；
* 2.如果在`xml`布局中声明了`fragment`标签，会导致模块之间极高的耦合。

什么叫做 **fragment标签会导致模块之间极高的耦合** ？举例来说，开发者在`layout`文件中声明这样一个`Fragment`:

```xml
<?xml version="1.0" encoding="utf-8"?>
<android.support.constraint.ConstraintLayout xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:app="http://schemas.android.com/apk/res-auto"
    xmlns:tools="http://schemas.android.com/tools"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    tools:context=".MainActivity">

    <!-- 声明一个fragment -->
    <fragment
        android:id="@+id/fragment"
        android:name="com.github.qingmei2.myapplication.AFragment"
        android:layout_width="match_parent"
        android:layout_height="match_parent"/>

</android.support.constraint.ConstraintLayout>
```

看起来似乎没有什么问题，但读者认真思考会发现，如果这是一个v4包的`Fragment`，是否意味着`LayoutInflater`额外增加了对`Fragment`类的依赖，类似这样：

```java
// LayoutInflater类
void rInflate(XmlPullParser parser, View parent, Context context, AttributeSet attrs) {
  // 1.解析当前控件
  while (parser.next()!= XmlPullParser.END_TAG) {
    //【注意】2.如果标签是一个Fragment，反射生成Fragment并返回
    if (name == "fragment") {
      Fragment fragment = clazz.newInstance();
      // .....还会关联到SupportFragmentManager、FragmentTransaction的依赖！
      supportFragmentManager.beginTransaction().add(....).commit();
      return;
    }

    final View view = createViewFromTag(parent, name, context, attrs);
    final ViewGroup viewGroup = (ViewGroup) parent;
    final ViewGroup.LayoutParams params = viewGroup.generateLayoutParams(attrs);
    // 3.解析子布局
    rInflateChildren(parser, view, attrs, true);
    // 所有子布局解析结束，将当前控件及布局参数添加到父布局中
    viewGroup.addView(view, params);
  }
}
```

这导致了`LayoutInflater`在解析`fragment`标签过程中，强制依赖了很多设计者不希望的依赖（比如v4包下`Fragment`相关类），继续往下思考的话，还会遇到更多的问题，这里不再引申。

那么如何解决这样的两个问题呢？

### 解决思路

考虑到 **性能优化** 和 **可扩展性**，设计者为`LayoutInflater`设计了一个`LayoutInflater.Factory`接口，该接口设计得非常巧妙：在`xml`解析过程中，开发者可以通过配置该接口对`View`的创建过程进行拦截：**通过new的方式创建控件以避免大量地使用反射**，亦或者 **额外配置特殊标签的解析逻辑以创建特殊组件** ：

```java
public abstract class LayoutInflater {
  private Factory mFactory;
  private Factory2 mFactory2;
  private Factory2 mPrivateFactory;

  public void setFactory(Factory factory) {
    //...
  }

  public void setFactory2(Factory2 factory) {
      // Factory 只能被set一次
      if (mFactorySet) {
          throw new IllegalStateException("A factory has already been set on this LayoutInflater");
      }
      mFactorySet = true;
      mFactory = mFactory2 = factory;
      // ...
  }

  public interface Factory {
    public View onCreateView(String name, Context context, AttributeSet attrs);
  }

  public interface Factory2 extends Factory {
    public View onCreateView(View parent, String name, Context context, AttributeSet attrs);
  }
}
```

正如上文所说的，`Factory`接口的意义是在`xml`解析过程中，开发者可以通过配置该接口对`View`的创建过程进行拦截，对于`View`的实例化，最终实现的伪代码如下：

```java
View createViewFromTag() {
  View view;
  // 1. 如果mFactory2不为空, 用mFactory2 拦截创建 View
  if (mFactory2 != null) {
      view = mFactory2.onCreateView(parent, name, context, attrs);
  // 2. 如果mFactory不为空, 用mFactory 拦截创建 View
  } else if (mFactory != null) {
      view = mFactory.onCreateView(name, context, attrs);
  } else {
      view = null;
  }

  // 3. 如果经过拦截机制之后，view仍然是null，再通过系统反射的方式，对View进行实例化
  if (view == null) {
      view = createView(name, null, attrs);
  }
}
```

理解了`LayoutInflater.Factory`接口设计的思路，接下来一起来思考如何解决上文中提到的2个问题。

### 减少反射次数

`AppCompatActivity`的源码中隐晦地配置`LayoutInflater.Factory`减少了大量反射创建控件的情况——设计者的思路是，在`AppCompatActivity`的`onCreate()`方法中，为`LayoutInflater`对象调用了`setFactory2()`方法：

```java
// AppCompatActivity类
@Override
protected void onCreate(@Nullable Bundle savedInstanceState) {
    getDelegate().installViewFactory();
    //...
}

// AppCompatDelegateImpl类
@Override
public void installViewFactory() {
    LayoutInflater layoutInflater = LayoutInflater.from(mContext);
    if (layoutInflater.getFactory() == null) {
      LayoutInflaterCompat.setFactory2(layoutInflater, this);
    }
}
```

配置之后，在`inflate()`过程中，系统的基础控件的实例化都通过代码拦截，并通过`new`的方式进行返回：


```java
switch (name) {
    case "TextView":
        view = new AppCompatTextView(context, attrs);
        break;
    case "ImageView":
        view = new AppCompatImageView(context, attrs);
        break;
    case "Button":
        view = new AppCompatButton(context, attrs);
        break;
    case "EditText":
        view = new AppCompatEditText(context, attrs);
        break;
    // ...
    // Android 基础组件都通过new方式进行创建
}
```

源码也说明了，即使开发者在`xml`文件中配置的是`Button`，`setContentView()`之后，生成的控件其实是`AppCompatButton`, `TextView`或者`ImageView`亦然，在避免额外的性能损失的同时，也保证了`Android`版本的向下兼容。

### 特殊标签的解析策略

为什么`Fragment`没有定义类似`void setContentView(R.layout.xxx)`的函数对布局进行填充，而是使用了`View onCreateView()`这样的函数，让开发者填充并返回一个对应的`View`呢？

原因就在于在布局填充的过程中，`Fragment`最终被视为一个子控件并添加到了`ViewGroup`中，设计者将`FragmentManagerImpl`作为`FragmentManager`的实现类，同时实现了`LayoutInflater.Factory2`接口。

而在布局文件中`fragment`标签解析的过程中，实际上是调用了`FragmentManagerImpl.onCreateView()`函数，生成了`Fragment`之后并将`View`返回，跳过了系统反射生成`View`相关的逻辑:

```java
# android.support.v4.app.FragmentManager$FragmentManagerImpl
@Override
public View onCreateView(View parent, String name, Context context, AttributeSet attrs) {
   if (!"fragment".equals(name)) {
       return null;
   }
   // 如果标签是`fragment`，生成Fragment，并返回Fragment的Root
   return fragment.mView;
}
```

通过定义`LayoutInflater.Factory`接口，设计者将`Fragment`的功能抽象为一个`View`（虽然`Fragment`并不是一个`View`），并交给`FragmentManagerImpl`进行处理，减少了模块之间的耦合，可以说是非常优秀的设计。

> 实际上`LayoutInflater.Factory`接口的设计还有更多细节（比如`LayoutInflater.FactoryMerger`类），篇幅原因，本文不赘述，有兴趣的读者可以研究一下。

## 小结

`LayoutInflater`整体的设计非常复杂且巧妙，从应用启动到进程间通信，从组件的启动再到组件UI的渲染，都可以看到`LayoutInflater`的身影，因此非常值得认真学习一番，建议读者参考本文开篇的思维导图并结合`Android`源码进行整体小结。

## 参考

* Android源码
* [Android探究LayoutInflater setFactory](https://blog.csdn.net/lmj623565791/article/details/51503977)
* [LayoutInflater——你应该知道的一点知识](https://www.jianshu.com/p/c29ec7e79d0d)

---

## 关于我

Hello，我是 [却把清梅嗅](https://github.com/qingmei2) ，如果您觉得文章对您有价值，欢迎 ❤️，也欢迎关注我的 [博客](https://juejin.im/user/588555ff1b69e600591e8462/posts) 或者 [Github](https://github.com/qingmei2)。

如果您觉得文章还差了那么点东西，也请通过**关注**督促我写出更好的文章——万一哪天我进步了呢？

* [我的Android学习体系](https://github.com/qingmei2/android-programming-profile)
* [关于文章纠错](https://github.com/qingmei2/Programming-life/blob/master/error_collection.md)
* [关于知识付费](https://github.com/qingmei2/Programming-life/blob/master/appreciation.md)
* [关于《反思》系列](https://github.com/qingmei2/android-programming-profile/blob/master/src/%E5%8F%8D%E6%80%9D%E7%B3%BB%E5%88%97/%E5%8F%8D%E6%80%9D%7C%E7%B3%BB%E5%88%97%E7%9B%AE%E5%BD%95.md)
