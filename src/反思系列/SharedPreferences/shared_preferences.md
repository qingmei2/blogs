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

不仅如此，`Android 8.0`前后的源码中，`SharedPreferences`内部的实现也略有不同。由此可见，
`Android`官方一直在尽力“拯救”`SharedPreferences`。

因此，在毅然决然抛弃`SharedPreferences`投奔新的解决方案之前，我们有必要重新认识一下它。

`SharedPreferences`是`Android`平台上 **轻量级的存储类**，用来保存`App`的各种配置信息，其本质是一个以 **键值对**（`key-value`）的方式保存数据的`XML`文件，其保存在`/data/data/shared_prefs`目录下。
