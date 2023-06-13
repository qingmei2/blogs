#  WorkManager 的设计与实现：系统概述

## 困境

作为一名 `Android` 开发者，即使你没有用过，也一定对 `WorkManager` 耳熟能详。

自2018年发布以来，作为 `Google` 官方推出的架构组件，它未像 `LiveData`、`ViewModel` 一样广泛应用。究其原因，一起来看 **官方** 当初对 `WorkManager` 的描述：

> **`WorkManager`** 用于执行可 **延迟**、**异步** 的后台任务。它提供了一个 **可靠**、**可调度** 的后台任务执行环境，可以处理 **即使在应用退出或设备重启后仍需要运行** 的任务。

快速提炼重点，你得到了什么?

> `WorkManager`，可以处理后台任务，这些任务哪怕手机重启也一定会执行?

看完简介，我的内心毫无波澜，"关机重启仍会执行" 的确很不错，but who cares ? 我根本用不到这些。

它给人的第一印象并不惊艳，甚至可以说 **平平无奇** ，无论是相亲市场还是技术领域，这都非常致命。

经过一系列的实践和研究后，回过头再看 `WorkManager`，笔者认为 `WorkManager` 是传统 `Android` 领域内学院派编程风格的代表作，是沧海遗珠。它为 **后台任务的处理和调度** 提供了一个优秀的解决方案。

即使如此，`WorkManager` 仍和 `Paging` 面临着同样的 **困境**: 难以推广、默默无闻。简单的项目用不到，复杂的项目经过若干年的沉淀，该领域早已应用了其它方案，学习和迁移成本过高，以至不被需要。

——时至今日，社区内除了若干 **使用简介** 和 **源码分析** 的博客单篇，我们仍很难找到其 **实战进阶** 或 **最佳实践** 的相关系列。

精兵如炬，困龙难飞。

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

当后台任务可能被延时执行，接下来思考下一个问题，如何保证任务执行的可靠性？

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

## 优先级 ≠ 加急

### 4.任务约束
