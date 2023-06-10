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

最终的目的，并非是让读者将 `WorkManager` 强行引入和使用，而是对 **后台任务的管理和调度** 有一个清晰的认知——即使从未使用过，在将来的某一天，遇到类似的业务诉求时，也能快速形成一个清晰的思路和方案。

##  一、筑基

想要构建一个优秀的 **后台任务的管理和调度的组件库**（后文简称 **后台任务库** ），需要依靠不断的迭代、优化和扩展，最终成为一个灵活、完善的工具。

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
  
  private final Executor mMainThreadExecutor = new Executor() {
    @Override
    public void execute(@NonNull Runnable command) {
        mMainThreadHandler.post(command);
    }
  };
  
  private final Executor mBackgroundExecutor = Executors.newFixedThreadPool(
                Math.max(2, Math.min(Runtime.getRuntime().availableProcessors() - 1, 4)),
                createDefaultThreadFactory(isTaskExecutor));
}
```

> 为提高可读性，本文的示例代码都有 **大幅精简** ，读者应尽量避免 "只见树木而不见森林"，以理解设计思想为主。

---
此外，作为设计者，后续也可考虑提供对 `RxJava`、协程等异步工具的支持。
