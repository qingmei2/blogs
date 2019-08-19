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

除此之外，考虑到 **性能优化** 和 **可扩展性**，设计者为`LayoutInlater`设计了一个`LayoutInflater.Factory2`接口，该接口设计得非常巧妙：在`xml`解析过程中，开发者可以通过配置该接口对`View`的创建过程进行拦截：**通过new的方式创建控件以避免大量地使用反射**，亦或者 **额外配置特殊标签的解析逻辑以创建特殊组件**（比如`Fragment`）。

> `LayoutInflater.Factory2`接口在`Android SDK`中的应用非常普遍，`AppCompatActivity`和`FragmentManager`就是最有力的体现，`LayoutInflater.inflate()`方法的理解虽然重要，但笔者窃以为`LayoutInflater.Factory2`的重要性与其相比不逞多让。

对于`LayoutInflater`整体不甚熟悉的开发者而言，本小节文字描述似乎晦涩难懂，且难免有是否过度设计的疑惑，但这些文字的本质却是布局填充流程整体的设计思想，**读者不应该将本文视为源码分析，而应该将自己代入到设计的过程中** 。

## 创建流程的设计与实现

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

每当一个组件（如`Activity`）被创建，都为其创建一个对应的`ContextImpl`实例，当组件需要获取系统服务时，则交给了`ContextImpl`去处理:

```java
// Activity 是 ContextWrapper的子类
class ContextWrapper extends Context {
    // 1.将 ContextImpl 作为成员进行存储
    public ContextWrapper(Context base) {
        mBase = base;
    }

    // 2.该成员即 ContextImpl
    Context mBase;

    // 3.系统服务的获取实际上是交给了ContextImpl
    @Override
    public Object getSystemService(String name) {
      return mBase.getSystemService(name);
    }
}
```
