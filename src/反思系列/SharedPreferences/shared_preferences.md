# 反思｜Google也无力回天？SharedPreferences的设计与实现

就在前几日，有幸拜读到 [HiDhl](https://juejin.im/user/2594503168898744) 的[文章](https://juejin.im/post/6881442312560803853)，继腾讯开源类似功能的`MMKV`之后，`Google`官方维护的 `Jetpack DataStore` 组件终于横空出世——值得玩味的是，这是否意味着无论是腾讯三方还是`Google`官方的角度，`SharedPreferences`都彻底被拉下王座?

无论是`MMKV`的支持者还是`DataStore`的拥趸，`SharedPreferences`似乎都不值一提；值得深思的是，笔者通过面试或者其它方式，和一些开行交流时，却遇到了以下的情形：

在谈及`SharedPreferences`和`MMKV`，大多数人都能对前者的 **缺陷**，以及后者性能上若干 **数量级的优势** 娓娓道来；但是，在针对前者的短板进行细节化的讨论时，往往却得不到更深入性的结果。

* 为什么`SharedPreferences`会有这些缺陷？
* `Google`的开发者团队中，有没有对这些缺陷做过改进的尝试？
* 为什么不惜推倒重来，推出新的`DataStore`组件来代替前者？
* 令`Google`工程师掣肘，时隔今日，这些缺陷依然存在的最根本性原因是什么？

笔者认为，在 **个人成长** 的角度上 ，对于新技术的应用，彻底 **说服自己** 比 **说服别人** 更有意义——而想要解除这些潜藏在内心最深处的困惑，就不得不从`SharedPreferences`本身的设计与实现讲起了。

本文大纲如下：

// TODO1

## 一、无力回天？SharedPreferences的前世今生

> `Google`尽力了，这仗没法打。

`Google`一直在尽力拯救`SharedPreferences`。

就在2019年的`Google I/O`大会上，官方推出了`Jetpack Security`组件，旨在保证文件和`SharedPreferences`的安全性——`SharedPreferences`的包装类，`EncryptedSharedPreferences`类隆重登场。

由此可见，`SharedPreferences`
