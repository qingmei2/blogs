# 反思｜官方也无力回天？Android SharedPreferences的设计与实现

## 起源

就在前几日，有幸拜读到 [HiDhl](https://juejin.im/user/2594503168898744) 的[文章](https://juejin.im/post/6881442312560803853)，继腾讯开源类似功能的`MMKV`之后，`Google`官方维护的 `Jetpack DataStore` 组件横空出世——这是否意味着无论是腾讯三方还是`Google`官方的角度，`SharedPreferences`都彻底告别了这个时代？

无论是`MMKV`的支持者还是`DataStore`的拥趸，`SharedPreferences`似乎都不值一提；值得深思的是，笔者通过面试或者其它方式，和一些同行交流时，却遇到了以下的情形：

在谈及`SharedPreferences`和`MMKV`，大多数人都能对前者的 **缺陷**，以及后者性能上若干 **数量级的优势** 娓娓道来；但是，在针对前者的短板进行细节化的讨论时，往往却得不到更深入性的结果，简单列举几个问题如下：

* `SharedPreferences`是如何保证线程安全的，其内部的实现用到了哪些锁？
* 进程不安全是否会导致数据丢失？
* 数据丢失时，其最终的屏障——文件备份机制是如何实现的？
* 如何实现进程安全的`SharedPreferences`？

除此之外，站在 **设计者的角度** 上，还有一些与架构相关，且同样值得思考的问题：

* 为什么`SharedPreferences`会有这些缺陷，如何对这些缺陷做改进的尝试？
* 为什么不惜推倒重来，推出新的`DataStore`组件来代替前者？
* 令`Google`工程师掣肘，时隔今日，这些缺陷依然存在的最根本性原因是什么？

而想要解除这些潜藏在内心最深处的困惑，就不得不从`SharedPreferences`本身的设计与实现讲起了。

本文大纲如下：

// TODO1

## 一、SharedPreferences的前世今生

我们知道，就在不久前2019年的`Google I/O`大会上，官方推出了`Jetpack Security`组件，旨在保证文件和`SharedPreferences`的安全性，`SharedPreferences`的包装类，[`EncryptedSharedPreferences`](https://developer.android.google.cn/reference/kotlin/androidx/security/crypto/EncryptedSharedPreferences)隆重登场。

不仅如此，`Android 8.0`前后的源码中，`SharedPreferences`内部的实现也略有不同。由此可见，`Android`官方一直在尽力“拯救”`SharedPreferences`。

因此，在毅然决然抛弃`SharedPreferences`投奔新的解决方案之前，我们有必要重新认识一下它。

### 1、设计与实现：建立基本结构

`SharedPreferences`是`Android`平台上 **轻量级的存储类**，用来保存`App`的各种配置信息，其本质是一个以 **键值对**（`key-value`）的方式保存数据的`xml`文件，其保存在`/data/data/shared_prefs`目录下。

对于21世纪初，那个`Android`系统诞生的时代而言，使用`xml`文件保存应用轻量级的数据绝对是一个不错的主意。那个时代的`json`才刚刚出生不久，虽然也渐渐成为了主流的 **轻量级数据交换格式** ，但是其更多的优势还是在于 **可读性**，这也是笔者猜测没有使用`json`而使用`xml`保存的原因之一。

现在我们为这个 **轻量级的存储类** 建立了最基础的模型，通过`xml`中的键值对，将对应的数据保存到本地的文件中。这样，每次读取数据时，通过解析`xml`文件，得到指定`key`对应的`value`；每次更新数据，也通过文件中`key`更新对应的`value`。

### 2、读操作的优化

通过这样的方式，虽然我们建立了一个最简单的 **文件存储系统**，但是性能实在不敢恭维，每次读取一个`key`对应的值都要重新对文件进行一次读的操作？显然需要尽量避免笨重的`I/O`操作。

因此设计者针对读操作进行了简单的优化，当`SharedPreferences`对象第一次通过`Context.getSharedPreferences()`进行初始化时，对`xml`文件进行一次读取，并将文件内所有内容（即所有的键值对）缓到内存的一个`Map`中，这样，接下来所有的读操作，只需要从这个`Map`中取就可以了：

```java
final class SharedPreferencesImpl implements SharedPreferences {
  private final File mFile;             // 对应的xml文件
  private Map<String, Object> mMap;     // Map中缓存了xml文件中所有的键值对
}
```

读者不禁会有疑问，虽然节省了`I/O`的操作，但另一个视角分析，当`xml`中数据量过大时，这种 **内存缓存机制** 是否会产生 **高内存占用** 的风险？

这也正是很多开发者诟病`SharedPreferences`的原因之一，那么，从事物的两面性上来看，**高内存占用** 真的是设计者的问题吗？

不尽然，因为`SharedPreferences`的设计初衷是数据的 **轻量级存储** ，对于类似应用的简单的配置项（比如一个`boolean`或者`int`类型），即使很多也并不会对内存有过高的占用；而对于复杂的数据（比如复杂对象反序列化后的字符串），开发者更应该使用类似`Room`这样的解决方案，而非一股脑存储到`SharedPreferences`中。

因此，相对于「`SharedPreferences`会导致内存使用过高」的说法，笔者更倾向于更客观的进行总结：

虽然 **内存缓存机制** 表面上看起来好像是一种 **空间换时间** 的权衡，实际上规避了短时间内频繁的`I/O`操作对性能产生的影响，而通过良好的代码规范，也能够避免该机制可能会导致内存占用过高的副作用，所以这种设计是 **值得肯定** 的。

### 3、写操作的优化

针对写操作，设计者同样设计了一系列的接口，以达到优化性能的目的。

我们知道对键值对进行更新是通过`mSharedPreferences.edit().putString().commit()`进行操作的——`edit()`是什么，`commit()`又是什么，为什么不单纯的设计初`mSharedPreferences.putString()`这样的接口？

设计者希望，在复杂的业务中，有时候一次操作会导致多个键值对的更新，这时，与其多次更新文件，我们更倾向将这些更新 **合并到一次写操作** 中，以达到性能的优化。

因此，对于`SharedPreferences`的写操作，设计者抽象出了一个`Editor`类，不管某次操作通过若干次调用`putXXX()`方法，更新了几个`xml`中的键值对，只有调用了`commit()`方法，最终才会真正写入文件：

```java
// 简单的业务，一次更新一个键值对
sharedPreferences.edit().putString().commit();

// 复杂的业务，一次更新多个键值对，仍然只进行一次IO操作（文件的写入）
Editor editor = sharedPreferences.edit();
editor.putString();
editor.putBoolean();
editor.putInt();
editor.commit();   // commit()才会更新文件
```

了解到这一点，读者应该明白，通过简单粗暴的封装，以达到类似`SPUtils.putXXX()`这种所谓代码量的节省，从而忽略了`Editor.commit()`的设计理念和使用场景，往往是不可取的，从设计上来讲，这甚至是一种 **倒退** 。

另外一个值得思考的角度是，本质上文件的`I/O`是一个非常重的操作，直接放在主线程中的`commit()`方法某些场景下会导致`ANR`（比如数据量过大），因此更合理的方式是应该将其放入子线程执行。

因此设计者还为`Editor`提供了一个`apply()`方法，用于异步执行文件数据的同步，并推荐开发者使用`apply()`而非`commit()`。

看起来`Editor`+`apply()`方法对写操作做了很大的优化，但更多的问题随之而来，比如子线程更新文件，必然会引发 **线程安全问题**；此外，`apply()`方法真的能够像我们预期的一样，能够避免`ANR`吗？答案是并不能，这个我们后文再提。

### 4、数据的更新 & 文件数量的权衡

随着业务复杂度的上升，需要面对新的问题是，`xml`文件中的数据量愈发庞大，一次文件的写操作成本也愈发高昂。

`xml`中数据是如何更新的？读者可以简单理解为 **全量更新** ——通过上文，我们知道`xml`文件中的数据会缓存到内存的`mMap`中，每次在调用`editor.putXXX()`时，实际上会将新的数据存入在`mMap`，当调用`commit()`或`apply()`时，最终会将`mMap`的所有数据全量更新到`xml`文件里。

由此可见，`xml`中数据量的大小，的确会对 **写操作** 的成本有一定的影响，因此，设计者更建议将 **不同业务模块的数据分文件存储** ，即根据业务将数据存放在不同的`xml`文件中。

因此，不同的`xml`文件应该对应不同的`SharedPreferences`对象，如果想要对某个`xml`文件进行操作，就通过传不同的文件标识符，获取对应的`SharedPreferences`：

```java
@Override
public SharedPreferences getSharedPreferences(String name, int mode) {
  // name参数就是文件名，通过不同文件名，获取指定的SharedPreferences对象
}
```

因此，当`xml`文件过大时，应该考虑根据业务，细分为若干个小的文件进行管理；但过多的小文件也会导致过多的`SharedPreferences`对象，不好管理且易混淆。实际开发中，开发者应根据业务的需要进行对应的平衡。

## 二、代码的不稳定因素

> `SharedPreferences`是线程安全的吗？

毫无疑问，`SharedPreferences`是线程安全的，但这只是对成品而言，对于我们目前的实现，显然还有一定的差距——那么，`Google`的设计者是如何保证`SharedPreferences`是线程安全的呢？
