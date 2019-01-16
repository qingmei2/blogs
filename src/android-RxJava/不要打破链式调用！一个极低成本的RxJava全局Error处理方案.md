## RxJava与CallbackHell

在正式铺展开本文内容之前，我们先思考一个问题：

> 你认为 RxJava 真的好用吗,它好用在哪？

CallbackHell，中文翻译为 **回调地狱**，在以往没有依赖`RxJava` + `Retrofit`进行网络请求的代码中，这种代码并不少见（比如`AsyncTask`），我曾有幸见识并维护了各种3层4层`AsyncTask`回调嵌套的项目——后来我一直拒绝阅读`AsyncTask`的源码，我想这应该是一个很重要的原因。

很感谢 [@prototypez](https://juejin.im/user/5b0a6f4251882538bc7779c2) 的 [《RxJava 沉思录》](https://juejin.im/post/5b8f536c5188255c352d3528) 系列的文章，我个人认为它是 **目前国内关于RxJava讲解最好的系列** ，作者列举了国内大多数文章中，关于RxJava好处的最常见的一些呼声：

* 用到了观察者模式
* 链式编程（一行代码实现XXX）
* 清晰且简洁的代码
* 避免了Callback Hell

不可否认，这些的确都是RxJava优秀的闪光点，但我认为这不是核心，正如 [这篇文章](https://juejin.im/post/5b8f5f0ee51d450ea52f6a37) 所说的,其更重要的意义在于：
> RxJava 给我们的**事件驱动型**编程带来了新的思路，`RxJava` 的 `Observable` **一下子把我们的维度拓展到了时间和空间两个维度**。

**事件驱动型编程**这个词很准确，现在我重新组织我的语言，”不要打破链式调用！“，这句话更应该说，不要破坏**RxJava事件驱动型**的编程思想。

### 你到底想说什么？

现在让我们回到文章的标题上，Android开发中，**网络请求的错误处理**一直是一个无法回避的需求，有了随着`RxJava` + `Retrofit`的普及，难免会遇到这个问题：

> [Android开发中 RxJava+Retrofit 全局网络异常捕获、状态码统一处理](https://blog.csdn.net/mq2553299/article/details/70244529)

这是我17年年初总结的一篇博客，那时我对于`RxJava`的理解比较有限，我阅读了网上很多前辈的博客，并总结了文中的这种方案，就是把全局的error处理放在`onError()`中，并将`Subscriber`包装成`MySubscriber`：

```Java
public abstract class MySubscriber<T> extends Subscriber<T> {
　// ...
   @Override
    public void onError(Throwable e) {
         onError(ExceptionHandle.handleException(e));  // ExceptionHandle中就是全局处理的逻辑，详情参考上方文章
    }

    public abstract void onError(ExceptionHandle.ResponeThrowable responeThrowable);
}

api.requestHttp()  //网络请求
      .subscribeOn(Schedulers.io())
      .observeOn(AndroidSchedulers.mainThread())
      .subscribe(new MySubscriber<Model>(context) {        // 包装了全局error处理逻辑的MySubscriber
          @Override
          public void onNext(Model model) { // ... }

          @Override
           public void onError(ExceptionHandle.ResponeThrowable throwable) {
                 // .......
            }
      });
```

这种解决方案于我当时看来没有问题，我认为这应该就是 **完美的解决方案** 了吧。

很快我就意识到了另外一个问题，就是这种方案成功地驱动我写出了 **RxJava版本的Callback Hell。**

### RxJava版本的Callback Hell

我不想你们笑话我的代码，因此我决定先不把它们抛出来，来看一个常见的需求：

> 请求一个API，如果发生异常，弹出一个Dialog，询问用户是否重试，如果重试，重新请求这个API。

让我们看看可能很多开发者 **第一直觉** 会写出的代码（为了保证代码不那么啰嗦，这里我使用了`Kotlin`）：

```kotlin
api.requestHttp()
    .subscribe(
          onNext = {
                // ...
          },
          onError = {
                  AlertDialog.Builder(context)    // 弹出一个dialog，提示用户是否重试
                    .xxxx
                    .setPositiveButton("重试") { _, _ ->  //  点击重试按钮，重新请求
                              api.requestHttp()                  
                                 .subscribe(
                                       onNext = {   ...   },
                                       onError = {    ...   }
                                  )
                    }
                    .setNegativeButton("取消") { _, _ -> // 啥都不做 }
                    .show()
          }
    )
```

瞧！我们写出了什么！

现在你也许明白了我当时的处境，`onError()`和`onComplete()`意味着这次订阅事件的终止，如果全局的异常处理都放在`onError()`中，接下来如果还有其他的需求（比如网络请求），就意味着你要在这个回调方法中再添加一层回调。

在一边高呼`RxJava` **链式调用简洁好用**和 **避免了CallbackHell** 时，我们将 **响应式编程** 扔到了一旁，然后继续 **按照日常的思维** 写着 **如出一辙的代码**。

如果你觉得这种操作完全可以接受，我们可以将需求升级一下：

> 如果发生异常，弹出dialog提示用户重试，这种dialog最多可弹出3次。

好的，如果说，最多重试一次，让代码额外增加了1层回调的嵌套（实际上是2层，Dialog的点击事件本身也是一层回调），那么最多重试3次，就是.....4层回调：

```kotlin
api.requestHttp()
    .subscribe(
          onNext = {
                // ...
          },
          onError = {
                      api.requestHttp()                  
                         .subscribe(
                               onNext = {
                                     // ...
                                },
                               onError = {                         
                                      api.requestHttp()                  
                                        .subscribe(
                                              onNext = {   ...   },
                                              onError = {    ...   }     // 还有一层
                                       )   
                               }
                        )
          }
    )
```

你可以说，我把这个请求封装成一个函数，然后每次只调用函数就行了，话虽如此，你依然不能否认这种 **CallbackHell** 并不优雅。

### 现在，如果有一种优雅的解决方案，那么这种方案最好有哪些优点？

 如有可能，我希望它能做到的是：

#### 1.轻量级

轻量级意味着 **较低的依赖成本**，如果一个工具库，它又要依赖若干个三方库，首先apk体积的急速膨胀就令人无法接受。

#### 2.灵活

灵活意味着 **更低的迁移成本**，我不希望，**添加** 或者 **移除** 这个工具令我的整个项目发生巨大的改动，甚至是重构。

如有可能，**不要在已有的业务逻辑代码上进行修改**。

#### 3.低学习成本

**低的学习成本** 可以让开发者更快的上手这个工具。

#### 4.可高度扩展

如有可能，请让这个工具库能够**为所欲为**。

这样看来，上文中通过继承的方式对全局error的处理方案，存在着一定的局限性，抛开令人**瞠目结舌**的回调地狱之外，**不能用lambda表达式** 就已经让我难以忍受。

## RxWeaver: 一个轻量且灵活的全局Error处理中间件

我花了一些时间开源了这个工具：

>  [RxWeaver: A lightweight and flexible error handler tools for RxJava2.](https://github.com/qingmei2/RxWeaver)

**Weaver** 翻译过来叫做 **织布鸟**，我最初的目的也正是让这个工具能够**对逻辑代码正确地组织**，达到实现**RxJava全局Error处理**的需求。


## 怎么用？可以做到什么程度？

> 为了代码的**足够简洁**，我选择使用Kotlin作为示范代码，我保证你可以看懂并理解它们——如果你的项目中适用的开发语言是`Java`，也请不用担心， [RxWeaver](https://github.com/qingmei2/RxWeaver) 同样提供了`Java`版本的依赖和示例代码，你可以在[这里](https://github.com/qingmei2/RxWeaver/tree/java)找到它。

RxWeaver的配置非常简单，你只需要配置好对应的`GlobalErrorTransformer`类，然后在需要处理error的网络请求代码中，通过`compose()`操作符，将`GlobalErrorTransformer`交给RxJava, 请注意，**仅仅需要一行代码**：

```kotlin
private fun requestHttp() {
        serviceManager.requestHttp()     // 网络请求
                .compose(RxUtils.handleGlobalError<UserInfo>(this))    // 加上这行代码
                .subscribeOn(Schedulers.io())
                .observeOn(AndroidSchedulers.mainThread())
                .subscribe( // ....)
}
```

`RxUtils.handleGlobalError<UserInfo>(this)`类似`Java`中的静态工具方法，它会返回一个对应`GlobalErrorTransformer`的一个实例——里面存储的是对应的error处理逻辑，这个类并不是 **RxWeaver** 的一部分，而是根据不同项目的不同业务，自己实现的一个类：

```kotlin
object RxUtils {

    fun  handleGlobalError(activity: FragmentActivity): GlobalErrorTransformer {
          // ....
    }
}
```

现在我们需要知道的是，**这样一行代码，可以做到什么样的程度**。

让我们从3个不同梯度的需求看看这个工具的韧性：

### 1.当接受到某种Error时，Toast对应的信息展示给用户

这是最常见的一种需求，当出现某种特殊异常（本案例以JSONException为例），我们会通过Toast提示这样的消息给用户：

> 全局异常捕获-Json解析异常！

```kotlin
fun test() {
        Observable.error(JSONException("JSONException"))
                .compose(RxUtils.handleGlobalError<UserInfo>(this))
                .subscribeOn(Schedulers.io())
                .observeOn(AndroidSchedulers.mainThread())
                .subscribe {
                    // ...
                }
}
```
毫无疑问，当没有加`compose(RxUtils.handleGlobalError<UserInfo>(this))`这行代码时，这次订阅的结果必然是弹出一个 “onError：xxxx”的 toast。

现在我们加上了compose的这行代码，让我们拭目以待：

![1.gif](http://upload-images.jianshu.io/upload_images/7293029-77a44d6bc15249a3?imageMogr2/auto-orient/strip)


看起来成功了，即使我们在`onError()`里面针对`Exception`做出了单独的处理，但是这个JSONException依然被全局捕获了，并弹出了一个额外的toast ：“全局异常捕获-Json解析异常！” 。

这似乎是一个很简单的需求，我们提升一点难度：

### 2.当接收到某种Error时，弹出Dialog

这次需求是：

> 若接收到一个`ConnectException`（连接异常）,我们让弹出一个dialog，这个dialog只会弹一次,若用户选择重试，重新请求API

又回到了上文中这个可能会引发 **Callback Hell** 的需求，我们疑问，如何保证 **Dialog和重试逻辑正确执行的同时，不打破Observable流的连续性（链式调用）**？

```kotlin
fun test2() {
  Observable.error(ConnectException())        // 这次我们把异常换成了`ConnectException`
                .compose(RxUtils.handleGlobalError<UserInfo>(this))
                .subscribeOn(Schedulers.io())
                .observeOn(AndroidSchedulers.mainThread())
                .subscribe {
                    // ...
                }
}
```

依然是熟悉的代码，这次我们把异常换成了`ConnectException`，我们直接看结果：

因为我们数据源是一个固定的`ConnectException`，因此我们无论怎么重试，必然都只会接收到`ConnectException`，这不重要，你发现没有，即使是一个复杂的需求（弹出dialog，用户选择后，决定是否重新请求这个流），**RxWeaver** 依然可以胜任。

![2.gif](http://upload-images.jianshu.io/upload_images/7293029-e7621eeb429de33b?imageMogr2/auto-orient/strip)

最后一个案例，让我们再来一个更复杂的。

### 3.当接收到Token失效的Error时，跳转login界面。

详细需求是：

> 当接收到Token失效的Error时，跳转login界面，用户重新登录成功后，返回初始界面，并重新请求API；如果用户登录失败或取消登录，弹出错误信息。

显然这个逻辑有点复杂了, 对于实现这个需求来讲，似乎不太现实，这次是否会束手无策呢？

```kotlin
fun test3() {
    Observable.error(TokenExpiredException())
                .compose(RxUtils.handleGlobalError<UserInfo>(this))
                .subscribeOn(Schedulers.io())
                .observeOn(AndroidSchedulers.mainThread())
                subscribe {
                    // ...
                }
}
```

这次我们把异常换成了`TokenExpiredException`(因为直接实例化一个`HttpException`过于复杂，所以我们自定义一个异常模拟代替它)，我们直接看结果：

![3.gif](http://upload-images.jianshu.io/upload_images/7293029-f396e5f307180d6a?imageMogr2/auto-orient/strip)

当然，无论怎么重试，数据源始终只会发射`TokenExpiredException`，但是我们成功实现了这个看似复杂的需求。

### 4. 我想说明什么？

我认为[RxWeaver](https://github.com/qingmei2/RxWeaver)达到了我心目中的设计要求：

* 轻量级

你不需要担心 **[RxWeaver](https://github.com/qingmei2/RxWeaver)**  的体积，它足够的轻量，轻量到所有类加起来只有不到200行代码，同时，除了`RxJava`和`RxAndroid`,它 **没有任何其它的依赖** ，体积大小只有3kb。

* 灵活

 **[RxWeaver](https://github.com/qingmei2/RxWeaver)** 的配置不需要 **修改** 或者 **删除** 任意一行已经存在的业务代码——它是完全可插拔的。

* 低学习成本

它的原理也是非常 **简单** 的，只要熟悉了`onErrorResumeNext`、`retryWhen`、`doOnError`这几个关键的操作符，你就可以马上上手对应的配置。

* 高扩展性

可以通过接口实现任意复杂的需求实现。

## 原理


> 这似乎本末倒置了，对于一个工具来说，**熟练使用API** 往往比 **阅读源码并了解原理** 优先级更高一些。但是我的想法是，如果你先了解了原理，这个工具的使用你会更加得心应手。

### **RxWeaver**的原理复杂吗？

实际上，RxWeaver的源码非常简单，简单到组件内部 **没有任何Error处理逻辑**，所有的逻辑都交给用户进行配置，它只是一个 **中间件**。

它的原理也是非常 **简单** 的，只要熟悉了`onErrorResumeNext`、`retryWhen`、`doOnError`这几个关键的操作符，你就可以马上上手对应的配置。

### 1.compose操作符

对于全局异常的处理，我只需要在既有代码的 **链式调用** 加上一行代码，配置一个 `GlobalErrorTransformer<T>`  交给 `compose()` 操作符————这个操作符是 `RxJava` 给我们提供的可以面向 **响应式数据类型** (Observable/Flowable/Single等等)进行 **AOP** 的接口, 可以对响应式数据类型 **加工** 、**修饰** ，甚至 **替换**。

这意味着，在既有的代码上，使用`compose()`操作符，我可以将一段特殊处理的逻辑代码插入到这个`Observable`中，这实在太方便了。

> 对compose操作符不了解的同学，请参考 [【译】避免打断链式结构：使用.compose()操作符 @by小鄧子](https://www.jianshu.com/p/e9e03194199e)

`compose()` 操作符需要我传入一个对应 **响应式类型**  (Observable/Flowable/Single等等)的`Transformer`接口，但是问题是不同的 **响应式类型** 对应不同的 `Transformer` 接口，不同的于是我们实现了一个通用的 `GlobalErrorTransformer<T>` 接口以 **兼容不同响应式类型的事件流** ：

```Kotlin
class GlobalErrorTransformer<T> constructor(
        private val globalOnNextRetryInterceptor: (T) -> Observable<T> = { Observable.just(it) },
        private val globalOnErrorResume: (Throwable) -> Observable<T> = { Observable.error(it) },
        private val retryConfigProvider: (Throwable) -> RetryConfig = { RetryConfig() },
        private val globalDoOnErrorConsumer: (Throwable) -> Unit = { },
        private val upStreamSchedulerProvider: () -> Scheduler = { AndroidSchedulers.mainThread() },
        private val downStreamSchedulerProvider: () -> Scheduler = { AndroidSchedulers.mainThread() }
) : ObservableTransformer<T, T>, FlowableTransformer<T, T>, SingleTransformer<T, T>,  MaybeTransformer<T, T>, CompletableTransformer {
      // ...
}
```

现在我们思考一下，如果我们想把error处理的逻辑放在`GlobalErrorTransformer`里面，把这个`GlobalErrorTransformer`交给`compose()` 操作符，就等于把error处理的逻辑全部 **插入** 到既有的`Observable`事件流中了:

```kotlin
fun test() {
    observable
          .compose(RxUtils.handleGlobalError<UserInfo>(this))   // 插入异常处理逻辑
          .subscribeOn(Schedulers.io())
          .observeOn(AndroidSchedulers.mainThread())
          subscribe {
              // ...
          }
}
```

同理，如果某个API不需要追加全局异常处理的逻辑，只需要把这行代码删掉即可，不会影响其他的业务代码。

这是一个不错的思路，接下来，我们需要思考的是，如何将不同的异常处理逻辑加进`GlobalErrorTransformer`中？

### 2.简单的全局异常处理：doOnError操作符

这个操作符的作用实在非常明显了，就是当我们接收到某个 **Throwable** 时，想要做的逻辑：

> ![image](http://upload-images.jianshu.io/upload_images/7293029-f446ce14603db6c0?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

这实在很适合大部分简单的错误处理需求，就像上文的需求1一样，当我们接收到某种指定的异常，弹出对应的message提示用户,逻辑代码如下：

```kotlin
when (error) {
    is JSONException -> {
        Toast.makeText(activity, "全局异常捕获-Json解析异常！", Toast.LENGTH_SHORT).show()
    }
    else -> {

    }
}
```

这种错误的处理方式， **不会对既有的Observable进行变换** ，也就是说，`JSONException` 依然会最终传递到subscribe的 `onError()` 的回调中——你依然需要实现 `onError()` 的回调，哪怕什么都不做，如有必要，再进行特殊的处理，否则会发生崩溃。

这种方式很简单，但是涉及复杂的需求就无能为力了，这时候我们就需要借助`onErrorResumeNext`操作符了。

### 3.复杂的异步Error处理：onErrorResumeNext操作符

以上文的需求2为例，若接收到一个指定的异常,我们需展示一个Dialog，提示用户是否重试—— 这种情况下，`doOnError`操作符明显无能为力，因为它不具有 **对Observable进行变换的能力**。

这时就需要 `onErrorResumeNext` 操作符上场了，它的作用是：当流的事件传递过程中发生了错误，我们可以将一个新的流交个 `onErrorResumeNext` 操作符，以保证事件流的继续传递。

> ![image](http://upload-images.jianshu.io/upload_images/7293029-514607ed543d2b4f?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

这是一个被严重低估的操作符，这个操作符意味着，只要你给一个`Observable<T>`的，就能继续往下传递事件，那么，这和需求中的 **展示一个Dialog供用户选择** 有关系吗？

当然有关系，我们只需要把Dialog的事件转换成对应的`Observable`即可：

```kotlin
object RxDialog {

    /**
     * 简单的示例，弹出一个dialog提示用户，将用户的操作转换为一个流并返回
     */
    fun showErrorDialog(context: Context,
                        message: String): Single<Boolean> {

        return Single.create<Boolean> { emitter ->
            AlertDialog.Builder(context)
                    .setTitle("错误")
                    .setMessage("您收到了一个异常:$message,是否重试本次请求？")
                    .setCancelable(false)
                    .setPositiveButton("重试") { _, _ -> emitter.onSuccess(true) }
                    .setNegativeButton("取消") { _, _ -> emitter.onSuccess(false) }
                    .show()
        }
    }
}
```

RxDialog的 `showErrorDialog()` 函数将会展示一个Dialog，返回值为一个 `Single<Boolean>` 的流,当用户点击 **确定** 按钮，订阅者会接收到一个 `true` 事件，反之，点击 **取消** 按钮，则会收到一个 `false` 事件。

> RxJava还能这么用？

当然，`RxJava`所代表的是一种响应式的编程范式，在刚接触RxJava的时候，我们都见过这样一种说法：RxJava 非常强大的一点便是 **异步**。

现在我们回过头来，**网络请求的数据流** 代表的是一种异步，难道 **弹出一个dialog，等待的用户选择结果** 难道不也是一种异步吗？

换句话说，**网络请求** 的流中事件意味着 **网络请求的结果**，那么上文中的 `Single<Boolean>` 代表着流中的事件是 ** Dialog的点击事件**。

其实RxJava发展的这些年来，Github上的RxJava扩展库层出不穷，比如`RxPermission`,`RxBinding`等等等等，前者是将 **权限请求** 的结果作为事件，交给了`Observable`进行传递；后者则是将  **View对应的事件 ** （比如点击事件，长按事件等等）交给了`Observable`。

回过头来，我们现在通过`RxDialog`创建了一个 **响应式的Dialog**，并获取到了用户的选择结果`Single<Boolean>`，接下来我们需要做的就只是根据`Single<Boolean>`中事件的值来判断 **是否重新请求网络数据** 了。

### 4.重试的处理：retryWhen操作符

RxJava提供了 `retryWhen()` 操作符，交给我们去处理是否重新执行流的订阅（本文中就是指重新进行网络请求）：

> ![image](http://upload-images.jianshu.io/upload_images/7293029-a949093ce7693565?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

篇幅所限，我不会针对这个操作符进行太多的讲解，关于 `retryWhen()` 操作符，请参考：

[【译】对RxJava中.repeatWhen()和.retryWhen()操作符的思考 by 小鄧子](https://www.jianshu.com/p/023a5f60e6d0)

继续上文的思路，我们到了Dialog对应的`Single<Boolean>`流，当用户选择后，实例化一个**RetryConfig** 对象，并把选择的结果`Single<Boolean>`交给了 `condition` 属性：

```kotlin
RetryConfig(condition = RxDialog.showErrorDialog(params))

data class RetryConfig(
        val maxRetries: Int = DEFAULT_RETRY_TIMES,  // 最大重试次数，默认1
        val delay: Int = DEFAULT_DELAY_DURATION,    // 重试延迟，默认1000ms
        val condition: () -> Single<Boolean> = { Single.just(false) }  // 是否重试
)
```

现在让我们来重新整理一下思路：

1.当用户接收到一个指定的异常时，弹出一个Dialog，其选择结果为`Single<Boolean>`；  
2.`RetryConfig` 内部存储了一个`Single<Boolean>`  的属性，这是一个决定了是否重试的函数；  
3.当用户选择了确认按钮，将`Single(true)`交给并实例化一个`RetryConfig` ，这意味着会重试，如果选择了取消，则为`Single(false)`，意味着不会重试。

### 5.似乎...完成了？

> 看来，仅仅需要这几个操作符，Error处理复杂的需求我们已经能够实现了？

的确如此，实际上，`GlobalErrorTransformer`内部的处理，也正是调用这几个操作符:

```kotlin
class GlobalErrorTransformer<T> constructor(
        private val globalOnNextRetryInterceptor: (T) -> Observable<T> = { Observable.just(it) },
        private val globalOnErrorResume: (Throwable) -> Observable<T> = { Observable.error(it) },
        private val retryConfigProvider: (Throwable) -> RetryConfig = { RetryConfig() },
        private val globalDoOnErrorConsumer: (Throwable) -> Unit = { },
        private val upStreamSchedulerProvider: () -> Scheduler = { AndroidSchedulers.mainThread() },
        private val downStreamSchedulerProvider: () -> Scheduler = { AndroidSchedulers.mainThread() }
) : ObservableTransformer<T, T>,
        FlowableTransformer<T, T>,
        SingleTransformer<T, T>,
        MaybeTransformer<T, T>,
        CompletableTransformer {

    override fun apply(upstream: Observable<T>): Observable<T> =
            upstream
                    .flatMap {
                        globalOnNextRetryInterceptor(it)
                    }
                    .onErrorResumeNext { throwable: Throwable ->
                        globalOnErrorResume(throwable)
                    }
                    .observeOn(upStreamSchedulerProvider())
                    .retryWhen(ObservableRetryDelay(retryConfigProvider))
                    .doOnError(globalDoOnErrorConsumer)
                    .observeOn(downStreamSchedulerProvider())

    // 其他响应式类型同理...                  
}                    
```

这也正是 **RxWeaver** 这个工具为什么如此 **轻量** 的原因，即使是 **核心类** **`GlobalErrorTransformer`** 也并没有更复杂的逻辑，仅仅是对几个操作符的组合使用而已。

此外的几个类，也无非是对重试逻辑接口的封装罢了。

### 6.如何实现界面的跳转？

看到这里，有的小伙伴可能已经有这样一个疑问了：

> 需求2中的，Dialog的逻辑我能够理解，那么，需求3中，Token失效，跳转login并返回重试是如何实现的？

实际上，无论是 **网络请求** ， 还是 **弹出Dialog** ， 亦或者 **跳转Login**，其终究只是一个 **事件的流** 而已，前者能通过接口返回一个 `Observble<T>` 或者 `Single<T>`, **跳转Login** 当然也可以：

```kotlin
class NavigatorFragment : Fragment() {

    fun startLoginForResult(activity: FragmentActivity): Single<Boolean> {
        // ....
    }
}
```

篇幅所限，本文不进行实现代码的展示，源码请参考[这里](https://github.com/qingmei2/RxWeaver/blob/kotlin/sample/src/main/java/com/github/qingmei2/model/NavigatorFragment.kt)。

其原理和 [RxPermissions](https://github.com/tbruyelle/RxPermissions) 、[RxLifecycle](https://github.com/trello/RxLifecycle) 还有笔者的 [RxImagePicker](https://github.com/qingmei2/RxImagePicker)   完全一样，依靠一个不可见的`Fragment` 对数据进行传递。

### 7.没看懂？

这是必然的，因为本文并没有向读者展示 **完整的代码**，原因有二，一是我认为庞大繁多的代码块会 **影响读者的体验**；二是， 不利于我阐述 **自己的设计思路** 。

我强烈建议你进入 **RxWeaver** 的官网，里面提供了本文所阐述所有的示例sample的源码：

> kotlin版：https://github.com/qingmei2/RxWeaver  
Java版：https://github.com/qingmei2/RxWeaver/tree/java

## 小结：RxJava，复杂还是简单

在本文的开始，我简单介绍了 [RxWeaver](https://github.com/qingmei2/RxWeaver) 的几个优点，其中一个是 **极低的学习成本**。

本文发布之前，我把我的工具介绍给了一些刚接触 **RxJava** 的开发者，他们接触之后，反馈竟然出奇的统一：

> 你这个东西太难了！

对于这个结果，我很诧异，因为这毕竟只是一个加起来还不到200行的工具库，后来我仔细的思考，我终于得出了一个结论，那就是：

> 本文的内容理解起来很 **简单** ，但首先需要你对RxJava有一定的理解，这比较 **困难**。

RxJava的学习曲线非常陡峭！正如 [@prototypez](https://juejin.im/user/5b0a6f4251882538bc7779c2) 在他的 [这篇文章](https://juejin.im/post/5b8f5f0ee51d450ea52f6a37) 中所说的一样：

> RxJava 是一个 “夹带了私货” 的框架，它本身最重要的贡献是提升了我们思考事件驱动型编程的维度，但是它与此同时又逼迫我们去接受了函数式编程。

正如本文一开始所说的，我们已经习惯了 **过程式编程** 的思维，因此文中的一些 **抽象的操作符** 会让我们陷入一定的迷茫，但是这也正是 **RxJava** 的魔力所在——它让我不断想要将新的需求 **从更高层级进行抽象**，尝试写出更简洁的代码（至少在我看来）。

我非常喜欢 **[RxWeaver](https://github.com/qingmei2/RxWeaver)** , 有朋友说说它代码有点少，但我却认为 **轻量** 是它最大的优点，它的本质目的也正是帮助开发者 **对业务逻辑进行组织，使其能够写出更 Reactive 和 Functional 的代码** 。
