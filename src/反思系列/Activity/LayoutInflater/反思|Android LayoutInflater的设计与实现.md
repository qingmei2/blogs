# 反思|Android LayoutInflater机制的设计与实现

> **反思** 系列博客是我的一种新学习方式的尝试，该系列起源和目录请参考 [这里](https://github.com/qingmei2/android-programming-profile/blob/master/src/%E5%8F%8D%E6%80%9D%E7%B3%BB%E5%88%97/%E5%8F%8D%E6%80%9D%7C%E7%B3%BB%E5%88%97%E7%9B%AE%E5%BD%95.md) 。

## 概述

`Android`本身体系非常宏大，源码中值得思考和借鉴之处众多。以`LayoutInflater`本身为例，其整个流程中除了调用`inflate()`函数 **填充布局** 功能之外，还分别涉及到了 **应用启动**、**调用系统服务**（进程间通信）、**对应组件作用域内单例管理**、**布局填充额外功能地扩展** 等等一系列的复杂逻辑需求。

本文笔者将针对`LayoutInlater`的整个设计思路进行描述，其整体结构如下图：

![Image1]()

## 整体思路

### 1、创建流程

顾名思义，`LayoutInflater`的作用就是 **布局填充器** ，其行为本质是调用了`Android`本身提供的 **系统服务**。而在`Android`系统的设计中，获取系统服务的实现方式就是通过通过`ServiceManager`来取得和对应服务交互的`IBinder`对象，然后创建对应系统服务的代理。

<!-- `Binder`机制相关并非本文的重点，读者需要了解的是，首先，为了保证布局填充的性能，在`Activity`创建并启动的过程中需要对`LayoutInlater`进行初始化，且单个`Activity`只会初始化一次，即 **作用域内的单例** 。其次，开发者只能通过`Activity`才能获取`LayoutInflater`？当然不，因此设计者需要将 **系统服务** 的获取通过抽象方法向上暴漏给`Context`，这意味着应用内其它组件（比如`Service`）也能获取对应的系统服务，而每种组件持有的`LayoutInlater`相对于组件而言同样是单例的。 -->

`Android`应用层将系统服务注册相关的`API`放在了`SystemServiceRegistry`类中，而将注册服务行为的代码放在了`ContextImpl`类中，`ContextImpl`类实现了`Context`类下的所有抽象方法。

`Android`应用层还定义了一个`Context`的另外一个子类：`ContextWrapper`，`Activity`、`Service`等组件继承了`ContextWrapper`, 每个`ContextWrapper`的实例有且仅对应一个`ContextImpl`，形成一一对应的关系，该类是 **装饰器模式** 的体现：保证了`Context`类公共功能代码和不同功能代码的隔离。

此外，虽然`ContextImpl`类作为`Context`类公共`API`的实现者，`LayoutInlater`的获取则交给了`ContextThemeWrapper`类，该类中将`LayoutInlater`的获取交给了一个成员变量，保证了单个组件 **作用域内的单例**。

### 2、布局填充流程

`API`的设计中，开发者可以直接调用`LayoutInflater#inflate()`函数对布局进行填充，该函数作用是对`xml`文件中标签的解析，并根据参数决定是否直接将新创建的`View`配置在指定的`ViewGroup`节点中。

一般来说，一个`View`的实例化依赖`Context`上下文对象和`attr`的属性集，而设计者正是通过将上下文对象和属性集作为参数，通过 **反射** 注入到`View`的构造器中对`View`进行创建。

此外，考虑到 **性能优化** 和 **可扩展性**，设计者为`LayoutInlater`设计了一个`LayoutInflater.Factory2`接口，该接口设计得非常巧妙：在`xml`解析过程中，开发者可以通过配置该接口对`View`的创建过程进行拦截：**通过new的方式创建控件以避免大量地使用反射**，亦或者 **额外配置特殊标签的解析逻辑以创建特殊组件**（比如`Fragment`）。

> `LayoutInflater.Factory2`接口在`Android SDK`中的应用非常普遍，`AppCompatActivity`和`FragmentManager`就是最有力的体现，`LayoutInflater.inflate()`方法的理解虽然重要，但`LayoutInflater.Factory2`同样值得花时间去细细品味。

对于`LayoutInflater`整体不甚熟悉的开发者而言，本小节文字描述似乎晦涩难懂，且难免有是否过度设计的疑惑，但这些文字的本质却是布局填充流程整体的设计思想，**读者不应该将本文视为源码分析，而应该将自己代入到设计的过程中** ，当深刻理解整个流程的设计思路之后，代码的设计和编写自然行云流水一气呵成。
