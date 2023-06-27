# 社区说|浅谈 WorkManager 的设计与实现：系统概述

> **反思** 系列博客是一种看似 **"内卷"** ，但却 **效果显著** 的学习方式，该系列起源和目录请参考 [这里](https://github.com/qingmei2/blogs/blob/master/src/%E5%8F%8D%E6%80%9D%E7%B3%BB%E5%88%97/thinking_in_android_index.md) 。

## 困境

作为一名 `Android` 开发者，即使你没有用过，也一定对 `WorkManager` 耳熟能详。

自2018年发布以来，作为 `Google` 官方推出的架构组件，它未像 `LiveData`、`ViewModel` 一样广泛应用。究其原因，一起来看 **官方** 当初对 `WorkManager` 的描述：

> **`WorkManager`** 用于执行可 **延迟**、**异步** 的后台任务。它提供了一个 **可靠**、**可调度** 的后台任务执行环境，可以处理 **即使在应用退出或设备重启后仍需要运行** 的任务。

快速提炼重点，我们得到了什么?

> `WorkManager`，可以处理后台任务，这些任务哪怕手机重启也一定会执行?

看完简介，我的内心毫无波澜，"关机重启仍会执行" 的确很不错，but who cares ? 我根本用不到这些。

它给人的第一印象并不惊艳，甚至可以说 **平平无奇** ，无论是相亲市场还是技术领域，这都非常致命。

经过一系列的实践和研究后，回过头再看 `WorkManager`，笔者认为 `WorkManager` 是传统 `Android` 领域内学院派编程风格的代表作，是沧海遗珠。它为 **后台任务的处理和调度** 提供了一个优秀的解决方案。

即使如此，`WorkManager` 仍和 `Paging` 面临着同样的 **困境**: 难以推广、默默无闻。简单的项目用不到，复杂的项目经过若干年的沉淀，该领域早已应用了其它方案，学习和迁移成本过高，以至不被需要。

——时至今日，社区内除了若干 **使用简介** 和 **源码分析** 的博客单篇，我们仍很难找到其 **实战进阶** 或 **最佳实践** 的相关系列。

## 目的

本文笔者将通过针对 `Android` 的后台任务管理机制，进行一个系统性的分析和设计。

最终的目的，并非是让读者将 `WorkManager` 强行引入和使用，而是对 **后台任务的管理和调度工具** （后文简称 **后台任务库** ）有一个清晰的认知——即使从未使用过，在将来的某一天，遇到类似的业务诉求时，也能快速形成一个清晰的思路和方案。

##  基本概念

想要构建一个优秀的后台任务库，需要依靠不断的迭代、优化和扩展，最终成为一个灵活、完善的工具。

那么 **后台任务库** 需要提供哪些功能，为什么要设计这些功能？

第一个映入眼帘的概念是：**线程调度**，顾名思义，它是后台异步任务的基石。

### 1.线程调度

举个例子，你负责的是一个视频类`APP`的研发，最初的业务诉求如下：

> `APP` 的日志上报。

需求清晰明了，显然，日志上报是一个 **后台异步任务**，在子线程进行 `IO` 操作： `Log` 文件本地的读写，以及上传到服务器。

这里我们引入 **任务执行器**（TaskExecutor）的概念：其用于执行后台任务的具体逻辑。通常，我们使用 **线程池** 来管理任务的线程，并提供线程的 **创建**、**销毁**、**调度** 等功能:

```java
// androidx.work.impl.utils.taskexecutor.WorkManagerTaskExecutor
public class WorkManagerTaskExecutor implements TaskExecutor {

  final Handler mMainThreadHandler = new Handler(Looper.getMainLooper());

  // 1.可切换主线程
  private final Executor mMainThreadExecutor = new Executor() {
    @Override
    public void execute(@NonNull Runnable command) {
        mMainThreadHandler.post(command);
    }
  };

  // 2.可切换后台工作线程
  private final Executor mBackgroundExecutor = Executors.newFixedThreadPool(
                Math.max(2, Math.min(Runtime.getRuntime().availableProcessors() - 1, 4)),
                createDefaultThreadFactory(isTaskExecutor));
}
```

> 为提高可读性，本文的示例代码都有 **大幅精简** ，读者应尽量避免「只见树木不见森林」，以理解设计理念为主。

这里我们为组件提供了最基础的线程调度的能力，便于内部实现 **主线程** 和 **后台线程** 的切换。其中我们为后台线程申请了一个线程池，并根据可用处理器核心数量来设置适当的线程数（通常同时最多执行的任务数为`4`），以充分利用设备的性能。

### 2.任务队列和串行化

接下来我们设计一个任务队列，保证可以不断接收新的任务，并及时分发给空闲的后台线程执行：

```java
public class ArrayDeque<E> extends AbstractCollection<E> implements Deque<E> {

  public boolean add(E e);

  public boolean offer(E e);

  public E remove();

  public E poll();
}
```

很经典的一个队列接口，`WorkManager` 也并未单独造轮子，而是借用了 `Java` 的 `ArrayDeque` 类。这是一个非常经典的实现类，在诸多大名鼎鼎的工具库中都有它的身影（比如`RxJava`)，限于篇幅不展开，感兴趣的读者可自行查看源码。

读者可预见到的是，后台任务的创建和添加的时机是无法控制的，加上 `ArrayDeque` 设计之初都并未考虑线程同步，因此目前的设计将会有 **线程安全问题** 。

因此，后台任务的入列和执行，必须借用一个新的角色保证串行，就像单线程一样。

实现方案非常简单，通过代理和加锁，提供一个`TaskExecutor`的包装类即可：

```java
public class SerialExecutorImpl implements SerialExecutor {

  // 1. 任务队列
  private final ArrayDeque<Task> mTasks;
  // 2. 后台线程的Executor，即上文中数量为4的线程池
  private final Executor mBackgroundExecutor;

  // 3.锁对象
  @GuardedBy("mLock")
  private Runnable mActive;

  @Override
  public void execute(@NonNull Runnable command) {
      // 不断地入列、出列和任务执行
      synchronized (mLock) {
          mTasks.add(new Task(this, command));
          if (mActive == null) {
            if ((mActive = mTasks.poll()) != null) {
                mExecutor.execute(mActive);
            }
          }
      }
  }
}
```

### 3.任务状态、类型和结果回调

下一步，我们对所关注的任务状态进行一个罗列，不难得出，我们关注的状态大致有：任务开始(Enqueued)、任务执行中（`Running`）、任务成功(`Successded`)、任务失败(`Failed`)、任务取消（`Cancelled`）等几种:

![](https://developer.android.google.cn/static/images/topic/libraries/architecture/workmanager/how-to/one-time-work-flow.png?hl=zh-cn)

请注意，上文中的日志上报我们定义成了 **一次性工作**，即只执行一次，执行结束即永久结束。

实际上，**一次性工作** 并不能涵盖所有的业务场景，举例来说，作为一个视频类的 `APP`，我们需 **定期** 对用户的播放进度进行一次记录或上报，保证用户即使杀掉`APP`，下次仍在最后记录的播放位置继续播放。

这里我们引入了 **定期任务** 的概念，它只有一个终止状态 `CANCELLED`。这是因为定期工作永远不会结束。每次运行后，无论结果如何，系统都会重新对其进行调度。

![](https://developer.android.google.cn/static/images/topic/libraries/architecture/workmanager/how-to/periodic-work-states.png?hl=zh-cn)

最后，我们将后台任务的执行抽象为一个接口，开发者实现 doWork 接口，实现具体后台业务，如日志上报等，并返回具体的结果：

```java
public abstract class Worker {

    // 后台任务的具体实现，`WorkerManager`内部进行了线程调度，执行在工作线程.
    @WorkerThread
    public abstract @NonNull Result doWork();
}

public abstract static class Result {

    @NonNull
    public static Result success() {
        return new Success();
    }

    @NonNull
    public static Result retry() {
        return new Retry();
    }

    @NonNull
    public static Result failure() {
        return new Failure();
    }
}
```

## 持久化

目前为止，我们实现了一个简化版、基于内存中队列的后台任务库，接下来我们将针对 **后台任务持久化** 的必要性，进行进一步的讨论。

### 1.非即时任务

第一步我们需要引入 **非即时任务** 的概念。

作为互联网的从业者，读者对类似的提示弹窗应该并不陌生：

> 您手机/PC的新版本已下载完毕：「立即安装」、「定时安装」、「稍后提醒我」

显然，这是一个常见的功能，用户选择后，应用或系统的后台需在未来的某个时间点，升级或提醒用户。如果和前文中的任务类型进行区分，前者我们可以归纳为 **即时任务**，后者则可称之为 **延时任务** 或 **非即时任务**。

非即时任务的诉求，将会使我们现有 **后台任务库** 的 **复杂度呈指数级提升** ，但这是 **必要** 的，原因如下：

首先，上文中 **定期任务** 也属于非即时任务的范畴，虽然该任务是立即执行并等待的，但实际上其真正的业务逻辑，仍是未来的某个时间点触发；其次，也是最重要的一点，作为一个健壮的后台任务库，和 **即时任务** 相比，对 **非即时任务** 提供支持的优先级要高得多。

——这似乎违反直觉，在日常开发中， **即时任务** 似乎才是主流，但我们忽视了一个事实，**资源并非无限**。

在文章的开始，我们构建了基本的线程调度能力，并创建了一个数量为 `4` 的线程池。但随着业务复杂度的提升，线程池可能会同时执行多个任务，这意味着部分晚入列、或优先级低的任务，会经常性等待前面的任务执行完毕。

严格意义上讲，此时，即时任务都转化为了非即时任务，再进一步抽象，**所有即时任务都是非即时任务**。

> **万物皆可异步**，是异步编程的一个经典概念，该思想在 `Handler`、`RxJava` 或 `协程` 中都有体现。

即时任务被延时执行是合理的吗？对于后台任务而言，是非常合理的，如果开发者有明确的诉求， **必须** 且 **立即** 执行某段业务逻辑，那么就不应该用 **后台任务库**，而是直接在内存中调用这块代码。

### 2.持久化存储

当后台任务可能被延时执行，思考下一个问题，如何保证任务执行的可靠性？

终极解决方案必然是 **后台任务持久化**，通过本地文件或者数据库存储后，即使进程被用户杀掉或系统回收，在合适的时机，`APP`总是能够将任务恢复并重建。

考虑到安全性，`WorkManager` 最终选择使用了 `Room` 数据库，并且设计维护了一个非常复杂的 `Database`，简单罗列下核心的 `WorkSpec` 表:

```java
@Entity(indices = [Index(value = ["schedule_requested_at"]), Index(value = ["last_enqueue_time"])])
data class WorkSpec(

    // 1.任务执行的状态，ENQUEUED/RUNNING/SUCCEEDED/FAILED/CANCELLED
    @JvmField
    @ColumnInfo(name = "state")
    var state: WorkInfo.State = WorkInfo.State.ENQUEUED,

    // 2.Worker的类名，便于反射和日志打印
    @JvmField
    @ColumnInfo(name = "worker_class_name")
    var workerClassName: String,

    // 3.输入参数
    @JvmField
    @ColumnInfo(name = "input")
    var input: Data = Data.EMPTY,

    // 4.输出参数
    @JvmField
    @ColumnInfo(name = "output")
    var output: Data = Data.EMPTY,

    // 5.定时任务
    @JvmField
    @ColumnInfo(name = "initial_delay")
    var initialDelay: Long = 0,

    // 6.定期任务
    @JvmField
    @ColumnInfo(name = "interval_duration")
    var intervalDuration: Long = 0,

    // 7.约束关系
    @JvmField
    @Embedded
    var constraints: Constraints = Constraints.NONE,

    // ...
)
```

设计好字段后，接下来我们设计其操作类 `WorkSpecDao` :

```java
@Dao
interface WorkSpecDao {
  // ...

  @Insert(onConflict = OnConflictStrategy.IGNORE)
  fun insertWorkSpec(workSpec: WorkSpec)

  @Query("SELECT * FROM workspec WHERE id=:id")
  fun getWorkSpec(id: String): WorkSpec?

  @Query("SELECT id FROM workspec")
  fun getAllWorkSpecIds(): List<String>

  @Query("UPDATE workspec SET state=:state WHERE id=:id")
  fun setState(state: WorkInfo.State, id: String): Int

  @Query("SELECT state FROM workspec WHERE id=:id")
  fun getState(id: String): WorkInfo.State?

  // ...
}
```

细心的读者会发现，`WorkSpecDao` 的设计中除了声明常规的 `Insert`、`Query`、`Update` 等，并没有 `DELETE` 类型的操作。

读者经过认真考虑后，可得该设计是合理的——由于有 `setState()` 可更新任务的状态，已完成或取消的工作无需删除，而是通过 `SQL` 语句，灵活分类按需获取，如：

```java
@Dao
interface WorkSpecDao {
  // ...

  // 获取全部执行中的任务
  @Query("SELECT * FROM workspec WHERE state=RUNNING")
  fun getRunningWork(): List<WorkSpec>

  // 获取全部近期已完成的任务
  @Query(
      "SELECT * FROM workspec WHERE last_enqueue_time >= :startingAt AND state IN COMPLETED_STATES ORDER BY last_enqueue_time DESC"
  )
  fun getRecentlyCompletedWork(startingAt: Long): List<WorkSpec>

  // ...
}
```

这可称之额外收获，通过 `Room` 的持久化存储，在保证了任务能够被稳定执行的同时，还可对所有任务进行备份，从而向开发者提供更多额外的能力。

> 准确来说，`WorkManager` 内部的 `Dao` 提供了 `Delete` 方法，但并未直接暴露给开发者，而是用于内部解决 **任务间的冲突** 问题，这个后文再提。

## 优先级管理

下面我们针对任务的 **优先级** 进一步进行讨论。

虽然上文明确说了，对于需要立即执行的行为，不应该作为后台任务，而是应该直接执行对应的业务代码块——看起来优先级机制并非刚需。

但实际上，这种机制仍然有一定的必要性。

### 1.约束条件

说到 **约束条件**，熟悉 `JobScheduler` 的开发者对此不会感到陌生，

约束条件 | 概念
--- | :---
NetworkType | 约束运行工作所需的网络类型。例如 Wi-Fi (UNMETERED)。
BatteryNotLow | 如果设置为 true，那么当设备处于“电量不足模式”时，工作不会运行。
RequiresCharging | 如果设置为 true，那么工作只能在设备充电时运行。
DeviceIdle | 如果设置为 true，则要求用户的设备必须处于空闲状态，才能运行工作。在运行批量操作时，此约束会非常有用；若是不用此约束，**批量操作可能会降低用户设备上正在积极运行的其他应用的性能。**
StorageNotLow | 如果设置为 true，那么当用户设备上的存储空间不足时，工作不会运行。

`WorkManager` 也提供了类似的概念，实际上内部也正是基于 `JobScheduler` 实现的，但 `WorkManager` 并非只是单纯的代理。

首先，当`API`版本不足时，`WorkManager` 兼容性使用 `AlarmManager` 或 `GcmScheduler` 作为补充：

```java
// androidx.work.impl.Schedulers.java
static Scheduler createBestAvailableBackgroundScheduler(
        @NonNull Context context,
        @NonNull WorkManagerImpl workManager) {

    Scheduler scheduler;

    if (Build.VERSION.SDK_INT >= WorkManagerImpl.MIN_JOB_SCHEDULER_API_LEVEL) {
        scheduler = new SystemJobScheduler(context, workManager);
    } else {
        scheduler = tryCreateGcmBasedScheduler(context);
        if (scheduler == null) {
            scheduler = new SystemAlarmScheduler(context);
        }
    }
    return scheduler;
}
```

其次，读者知道，`JobScheduler` 的 `onStartJob` 等回调默认运行在主线程，不能直接进行耗时操作。`WorkManager` 的 `Worker` 内部则进行了线程调度，默认实现在 `WorkerThread`：

```java
// androidx.work.impl.background.systemjob.SystemJobService
public class SystemJobService extends JobService {

    public boolean onStartJob(@NonNull JobParameters params) {
      //...
      mWorkManagerImpl.startWork(...);
    }
}

// androidx.work.impl.WorkManagerImpl
public class WorkManagerImpl extends WorkManager {
  public void startWork(...) {
      mWorkTaskExecutor.executeOnTaskThread(new StartWorkRunnable(...));
  }
}
```

由于 `JobScheduler` 是由系统服务中的 `JobSchedulerService` 实现的，因此其自身的高权限，可以在`APP`被杀或重启后，仍然可以唤起并执行 `JobService` 及对应的任务。

### 2.新的保活机制？

`Android` 官方提供了后台作业的强大支持，国内厂商大多数第一时间却想拿它来做 **保活**。

——举例来说，既然 `WorkManager` 支持定期任务，且即使 `APP` 被杀或者重启都能够保证执行；那么我一个 `IM APP`，每 `10` 秒拉取下接口看有没有新消息，顺便启动下 `APP` 某些页面或者通知组件，想必也是非常合理的。

![](jetpack_workmanager1.jpeg)

实际上，对于 **保活** 的诉求，`WorkManager` 做不到，其本质是 `JobScheduler` 做不到：

首先，随着版本的迭代，`Android` 系统对后台任务的管理愈发严苛，小于 `15` 分钟的定期任务已经被强制调整为 `15` 分钟执行，避免频繁的后台定时任务对前台应用的影响，规避了 `API` 的非法滥用：

> WorkManager : 我把你当兄弟，你竟然想睡我？

其次是笔者的猜测，由于用户安全等相关的考量，国内设备厂商对 `JobSchedulerService` 等相关都有一定的魔改——比如，当用户手动将 `APP` 强制关闭，这种操作意图拥有最高优先级，即使是系统服务也不应对其再次启动（厂商白名单除外，如微信和`QQ`?）。

两点结合，`WorkManager` 的定期任务受到了严格的限制，这也意味着类似**保活**需求其无法满足，其 “不稳定” 性这也是其国内应用较少的主要原因之一。

### 3.前台服务

最后，我们讨论下 **如何规范地调度系统资源** 。

最经典的场景仍然是后台任务的加急，即使有约束条件，部分后台任务仍需要被 **特殊加急** ，比如 **用户聊天时发送短视频**、**处理付款或订阅流程** 等。这些任务对用户很重要，会在后台快速执行，并需要立即开始执行。

> 系统对于应用的资源分配非常严格，读者可以通过 **[这里](https://developer.android.google.cn/topic/performance/appstandby?hl=zh-cn)** 简单了解。

由于任务的执行依赖于系统对资源的分配，因此想要提高执行的优先级，势必需要提升 `APP` 组件自身的优先级，那么实现方案已经非常明显了：使用 **前台服务** 。

当你需要通过调用类似 `worker.setExpedited(true)` 标记为加急任务时，`Worker` 内部的具体实现，需开发者额外创建前台通知，提升优先级的同时，将你后台的任务行为同步给用户。

## 小结

小结并不是总结，还有更多内容可以扩展，比如：

* 1、`Android` 系统的 `JobScheduler` 任务调度机制的内部原理是什么样的？

* 2、`WorkManager` 组件的初始化机制、持久化机制等，内部的实现细节有哪些需要注意的？

* 3、`WorkManager` 中的约束任务和非约束任务在执行上有什么不同？

* 4、`WorkManager` 还有哪些其它的亮点？

篇幅原因，这些问题笔者将另起一篇进行更深入性的讨论，敬请期待。

---

## 关于我

Hello，我是 [却把清梅嗅](https://github.com/qingmei2) ，如果您觉得文章对您有价值，欢迎 ❤️，也欢迎关注我的 [博客](https://blog.csdn.net/mq2553299) 或者 [GitHub](https://github.com/qingmei2)。

如果您觉得文章还差了那么点东西，也请通过 **关注** 督促我写出更好的文章——万一哪天我进步了呢？

* [我的Android学习体系](https://github.com/qingmei2/blogs)
* [关于文章纠错](https://github.com/qingmei2/blogs/blob/master/error_collection.md)
* [关于知识付费](https://github.com/qingmei2/blogs/blob/master/appreciation.md)
* [关于《反思》系列](https://github.com/qingmei2/blogs/blob/master/src/%E5%8F%8D%E6%80%9D%E7%B3%BB%E5%88%97/thinking_in_android_index.md)
