# 反思｜官方也无力回天？Android SharedPreferences的设计与实现

> **反思** 系列博客是我的一种新学习方式的尝试，该系列起源和目录请参考 [这里](https://github.com/qingmei2/blogs/blob/master/src/%E5%8F%8D%E6%80%9D%E7%B3%BB%E5%88%97/thinking_in_android_index.md) 。

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

![](https://img-blog.csdnimg.cn/20201017163353160.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L21xMjU1MzI5OQ==,size_16,color_FFFFFF,t_70#pic_center)

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
editor.putBoolean().putInt();
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

## 二、线程安全问题

> `SharedPreferences`是线程安全的吗？

毫无疑问，`SharedPreferences`是线程安全的，但这只是对成品而言，对于我们目前的实现，显然还有一定的差距，如何保证线程安全呢？

——那，为了保证线程安全，怎么着不得加个锁吧。

加个锁？那是起步！3把锁，你还别嫌多。你得研究开发写代码时的心理，舍得往代码里吭哧吭哧加锁的开发，压根不在乎再加2把。

### 1、保证复杂流程代码的可读性

为了保证`SharedPreferences`是线程安全的，`Google`的设计者一共使用了3把锁：

```java
final class SharedPreferencesImpl implements SharedPreferences {
  // 1、使用注释标记锁的顺序
  // Lock ordering rules:
  //  - acquire SharedPreferencesImpl.mLock before EditorImpl.mLock
  //  - acquire mWritingToDiskLock before EditorImpl.mLock

  // 2、通过注解标记持有的是哪把锁
  @GuardedBy("mLock")
  private Map<String, Object> mMap;

  @GuardedBy("mWritingToDiskLock")
  private long mDiskStateGeneration;

  public final class EditorImpl implements Editor {
    @GuardedBy("mEditorLock")
    private final Map<String, Object> mModified = new HashMap<>();
  }
}
```

对于这样复杂的类而言，如何提高代码的可读性？`SharedPreferencesImpl`做了一个很好的示范：**通过注释明确写明加锁的顺序，并为被加锁的成员使用`@GuardedBy`注解**。

对于简单的 **读操作** 而言，我们知道其原理是读取内存中`mMap`的值并返回，那么为了保证线程安全，只需要加一把锁保证`mMap`的线程安全即可：

```java
public String getString(String key, @Nullable String defValue) {
    synchronized (mLock) {
        String v = (String)mMap.get(key);
        return v != null ? v : defValue;
    }
}
```

那么，对于 **写操作** 而言，我们也能够通过一把锁达到线程安全的目的吗？

### 2、保证写操作的线程安全

对于写操作而言，每次`putXXX()`并不能立即更新在`mMap`中，这是理所当然的，如果开发者没有调用`apply()`方法，那么这些数据的更新理所当然应该被抛弃掉，但是如果直接更新在`mMap`中，那么数据就难以恢复。

因此，`Editor`本身也应该持有一个`mEditorMap`对象，用于存储数据的更新；只有当调用`apply()`时，才尝试将`mEditorMap`与`mMap`进行合并，以达到数据更新的目的。

因此，这里我们还需要另外一把锁保证`mEditorMap`的线程安全，笔者认为，不和`mMap`公用同一把锁的原因是，在`apply()`被调用之前，`getXXX`和`putXXX`理应是没有冲突的。

代码实现参考如下：

```java
public final class EditorImpl implements Editor {
  @Override
  public Editor putString(String key, String value) {
      synchronized (mEditorLock) {
          mEditorMap.put(key, value);
          return this;
      }
  }
}
```

而当真正需要执行`apply()`进行写操作时，`mEditorMap`与`mMap`进行合并，这时必须通过2把锁保证`mEditorMap`与`mMap`的线程安全，保证`mMap`最终能够更新成功，最终向对应的`xml`文件中进行更新。

文件的更新理所当然也需要加一把锁：

```java
// SharedPreferencesImpl.EditorImpl.enqueueDiskWrite()
synchronized (mWritingToDiskLock) {
    writeToFile(mcr, isFromSyncCommit);
}
```

最终，我们一共通过使用了3把锁，对整个写操作的线程安全进行了保证。

> 篇幅限制，本文不对源码进行详细引申，有兴趣的读者可参考 `SharedPreferencesImpl.EditorImpl` 类的`apply()`源码。

### 3、摆脱不掉的ANR

`apply()`方法设计的初衷是为了规避主线程的`I/O`操作导致`ANR`问题的产生，那么，`ANR`的问题真得到了有效的解决吗？

并没有，在 **字节跳动技术团队** 的 [这篇文章](https://mp.weixin.qq.com/s?__biz=MzI1MzYzMjE0MQ==&mid=2247484387&idx=1&sn=e3c8d6ef52520c51b5e07306d9750e70&scene=21#wechat_redirect) 中，明确说明了线上环境中，相当一部分的`ANR`统计都来自于`SharedPreference`，由此可见，`apply()`并没有完全规避掉这个问题，那么导致`ANR`的原因又是什么呢。

经过我们的优化，`SharedPreferences`的确是线程安全的，`apply()`的内部实现也的确将`I/O`操作交给了子线程，可以说其本身是没有问题的，而其原因归根到底则是`Android`的另外一个机制。

在`apply()`方法中，首先会创建一个等待锁，根据源码版本的不同，最终更新文件的任务会交给`QueuedWork.singleThreadExecutor()`单个线程或者`HandlerThread`去执行，当文件更新完毕后会释放锁。

但当`Activity.onStop()`以及`Service`处理`onStop`等相关方法时，则会执行 `QueuedWork.waitToFinish()`等待所有的等待锁释放，因此如果`SharedPreferences`一直没有完成更新任务，有可能会导致卡在主线程，最终超时导致`ANR`。

> 什么情况下`SharedPreferences`会一直没有完成任务呢？ 比如太频繁无节制的`apply()`，导致任务过多，这也侧面说明了`SPUtils.putXXX()`这种粗暴的设计的弊端。

`Google`为何这么设计呢？字节跳动技术团队的这篇文章中做出了如下猜测：

> 无论是 commit 还是 apply 都会产生 ANR，但从 Android 之初到目前 Android8.0，Google 一直没有修复此 bug，我们贸然处理会产生什么问题呢。Google 在 Activity 和 Service 调用 onStop 之前阻塞主线程来处理 SP，我们能猜到的唯一原因是尽可能的保证数据的持久化。因为如果在运行过程中产生了 crash，也会导致 SP 未持久化，持久化本身是 IO 操作，也会失败。

如此看来，导致这种缺陷的原因，其设计也的确是有自身的考量的，好在 [这篇文章](https://mp.weixin.qq.com/s?__biz=MzI1MzYzMjE0MQ==&mid=2247484387&idx=1&sn=e3c8d6ef52520c51b5e07306d9750e70&scene=21#wechat_redirect) 末尾也提出了一个折衷的解决方案，有兴趣的读者可以了解一下，本文不赘述。

## 三、进程安全问题

### 1、如何保证进程安全

`SharedPreferences`是否进程安全呢？让我们打开`SharedPreferences`的源码，看一下最顶部类的注释：

```java
/**
 * ...
 * This class does not support use across multiple processes.
 * ...
 */
public interface SharedPreferences {
  // ...
}
```

由此，由于没有使用跨进程的锁，`SharedPreferences`是进程不安全的，在跨进程频繁读写会有数据丢失的可能，这显然不符合我们的期望。

那么，如何保证`SharedPreferences`进程的安全呢?

实现思路很多，比如使用文件锁，保证每次只有一个进程在访问这个文件；或者对于`Android`开发而言，`ContentProvider`作为官方倡导的跨进程组件，其它进程通过定制的`ContentProvider`用于访问`SharedPreferences`，同样可以保证`SharedPreferences`的进程安全；等等。

> 篇幅原因，对实现有兴趣的读者，可以参考 **百度** 或文章末尾的 **参考资料**。

### 2、文件损坏 & 备份机制

`SharedPreferences`再次迎来了新的挑战。

由于不可预知的原因（比如内核崩溃或者系统突然断电），`xml`文件的 **写操作** 异常中止，`Android`系统本身的文件系统虽然有很多保护措施，但依然会有数据丢失或者文件损坏的情况。

作为设计者，如何规避这样的问题呢？答案是对文件进行备份，`SharedPreferences`的写入操作正式执行之前，首先会对文件进行备份，将初始文件重命名为增加了一个`.bak`后缀的备份文件：

```java
// 尝试写入文件
private void writeToFile(...) {
  if (!backupFileExists) {
      !mFile.renameTo(mBackupFile);
  }
}
```

这之后，尝试对文件进行写入操作，写入成功时，则将备份文件删除：

```java
// 写入成功，立即删除存在的备份文件
// Writing was successful, delete the backup file if there is one.
mBackupFile.delete();
```

反之，若因异常情况（比如进程被杀）导致写入失败，进程再次启动后，若发现存在备份文件，则将备份文件重名为源文件，原本未完成写入的文件就直接丢弃：

```java
// 从磁盘初始化加载时执行
private void loadFromDisk() {
    synchronized (mLock) {
        if (mBackupFile.exists()) {
            mFile.delete();
            mBackupFile.renameTo(mFile);
        }
    }
  }
```

现在，通过文件备份机制，我们能够保证数据只会丢失最后的更新，而之前成功保存的数据依然能够有效。

## 四、小结

综合来看，`SharedPreferences`那些一直被关注的问题，从设计的角度来看，都是有其自身考量的。

我们可以看到，虽然`SharedPreferences`其整体是比较完善的，但是为什么相比较`MMKV`和`Jetpack DataStore`，其性能依然有明显的落差呢？

这个原因更加综合且复杂，即使笔者也还是处于浅显的了解层面，比如后两者在其数据序列化方面都选用了更先进的`protobuf`协议，`MMKV`自身的数据的 **增量更新** 机制等等，有机会的话会另起新的一篇进行分享。

反过头来，相对于对组件之间单纯进行 **好** 和 **不好** 的定义，笔者更认为通过辩证的方式去看待和学习它们，相信即使是`SharedPreferences`，学习下来依然能够有所收获。

## 参考 & 感谢

> 细心的读者应该能够发现，关于 **参考&感谢** 一节，笔者着墨越来越多，原因无他，笔者 **从不认为** 一篇文章就能够讲一个知识体系讲解的面面俱到，本文亦如是。
>
> 因此，读者应该有选择性查看其它优质内容的权利，甚至是为其增加一些简洁的介绍（因为标题大多都很相似），而不是文章末尾甩一堆`https`开头的链接不知所云。
>
> 这也是对这些内容创作者的尊重，如果你喜欢本文，也同样希望你能够喜欢下面这些文章。

[1、请不要滥用SharedPreference @Weishu](http://weishu.me/2016/10/13/sharedpreference-advices/)

我们如何定义好的文章？**深度** 和 **引人入胜**，笔者觉得缺一不可，深度保证了文章能够经久不衰，引人入胜代表了流畅的 **文字功底** 和 **文章结构**，这篇文章将`apply()`导致的`ANR`原理通过浅显易懂的方式解构的非常透彻，我认为它是最适合进阶学习`SharedPreferences`的文章。

[2、Android源码分析之SharedPreferences @xiaoweiz](https://www.cnblogs.com/xiaoweiz/p/3733272.html)

对于一门技术，如何系统掌握其 **理论** ，笔者的理解是，学习理解其设计思想，从零开始一步步完善整个系统解构，最终通过源码进行互相印证。

而对于`SharedPreferences`，学习设计思想，看本文；源码解析，看这篇。

[3、Android 之不要滥用 SharedPreferences（下） @godliness](https://www.jianshu.com/p/f5a29bce2e6f)

标题和1很相似，但内容更有深度，该文针对 **多进程下的文件安全问题** 和 **文件备份机制** 进行了源码级别的解析，值得收藏。

[4、剖析 SharedPreference apply引起的ANR问题 @字节跳动技术团队](https://mp.weixin.qq.com/s?__biz=MzI1MzYzMjE0MQ==&mid=2247484387&idx=1&sn=e3c8d6ef52520c51b5e07306d9750e70&scene=21#wechat_redirect)

针对`apply()`方法导致的`ANR`的问题，进行了原因定位和解决方案，非常值得阅读。

[5、再见 SharedPreferences 拥抱 Jetpack DataStore @HiDhl](https://juejin.im/post/6881442312560803853)

最近笔者非常关注的博主，文章都很有深度，文章中根据`SharedPreferences`的缺陷都进行了系统性的阐述，也是因为该文，引发了笔者写本文缅怀`SharedPreferences`的想法。

[6、通过ContentProvider实现SharedPreferences进程共享数据 @king龙123](https://www.jianshu.com/p/3e551e3d4a8d)

[7、Android使用读写锁实现多进程安全的SharedPreferences @痕迹丶](https://blog.csdn.net/qq_27512671/article/details/101445642)

针对保证`SharedPreferences`多进程安全的实现方案，有兴趣的读者可以作为引申阅读。


---

## 关于我

Hello，我是 [却把清梅嗅](https://github.com/qingmei2) ，如果您觉得文章对您有价值，欢迎 ❤️，也欢迎关注我的 [博客](https://blog.csdn.net/mq2553299) 或者 [GitHub](https://github.com/qingmei2)。

如果您觉得文章还差了那么点东西，也请通过 **关注** 督促我写出更好的文章——万一哪天我进步了呢？

* [我的Android学习体系](https://github.com/qingmei2/blogs)
* [关于文章纠错](https://github.com/qingmei2/blogs/blob/master/error_collection.md)
* [关于知识付费](https://github.com/qingmei2/blogs/blob/master/appreciation.md)
* [关于《反思》系列](https://github.com/qingmei2/blogs/blob/master/src/%E5%8F%8D%E6%80%9D%E7%B3%BB%E5%88%97/thinking_in_android_index.md)
