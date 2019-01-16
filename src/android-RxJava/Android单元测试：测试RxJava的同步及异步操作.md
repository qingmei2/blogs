##  简述

在您开发的项目中，您使用了RxJava+Retrofit的网络请求框架，在RxJava强大的操作符下，您理所当然的认为您已经能够游刃有余地处理Android客户端开发中的联网请求需求，比如这样：

```java
//Model层的网络请求
public class HomeModel extends BaseModel<ServiceManager> implements HomeContract.Model {

 @Override
 public Maybe<UserInfo> requestUserInfo(final String userName) {
      return serviceManager.getUserInfoService()
              .getUserInfo(userName)
              .subscribeOn(Schedulers.io())
              .observeOn(AndroidSchedulers.mainThread());
  }
}

//Presenter层的网络处理
public class HomePresenter extends BasePresenter<HomeContract.View, HomeContract.Model> implements HomeContract.Presenter {

    @Inject
    public HomePresenter(HomeContract.View rootView, HomeContract.Model model) {
        super(rootView, model);
    }

    @SuppressLint("VisibleForTests")
    @Override
    public void requestUserInfo(String userName) {
        mModel.requestUserInfo(userName)
                .subscribe(info -> Optional.ofNullable(info)
                                .map(UserInfo::getLogin)
                                .ifPresentOrElse(image -> mRootView.onGetUserInfo(info)
                                        , () -> mRootView.onError("用户信息为空"))
                        , e -> mRootView.onError("请求出现异常"));
    }
}

```
 
 这是一个非常简单的MVP开发模式下的一个联网请求代码，用于请求用户的信息数据，我们不需要强制性理解每一行代码，我们需要关注的问题是，我们如何保证这些代码的正确性，即使在一些边界条件的限制下？
 
单元测试能够让项目中自己的代码更加健壮，但是无论是RxJava+Retrofit,亦或是okHttp3，异步操作的测试都是一个较高的门槛，加上目前国内开发环境的恶劣环境（周期短，需求变动反复），国内的相关技术文档相当匮乏。

## Espresso的异步测试

笔者曾经花费一些时间研究Espresso的异步测试，并根据google官方的demo尝试总结在设备上进行测试异步操作的方案：

[Android 自动化测试 Espresso篇：异步代码测试](http://blog.csdn.net/mq2553299/article/details/74490718)

这是可行的，但是有一些问题需要注意：
 
 1.Espresso本质是依赖设备所进行的集成测试，这种测试是业务级的，测试的内容是一个整体的功能，相对于代码（方法）级别的单元测试，覆盖范围太大。
 2.Espresso的测试依赖设备，所以进行一次完整的测试所消耗的时间更长。
 3.Espresso更擅长UI的测试，而对于业务代码来说，Espresso并不擅长。

现在我们需要一种新的方式去测试业务代码，比如Junit+Mockito.

## 异步测试的难题

在尝试RxJava的单元测试时，不可避免的，你会遇到两个问题：

1.如何测试异步代码？
2.如何测试AndroidSchedulers.mainThread()？

### 1.异步操作如何转为同步进行单元测试？
 
 前者的问题我们已经在之前的[Android 自动化测试 Espresso篇：异步代码测试](http://blog.csdn.net/mq2553299/article/details/74490718)这篇文章中提到过了，那就是：

> 在我们还没有获取到数据进行验证时，测试代码已经跑完了。

以下面一段代码为例：
```
//开启一个子线程，每隔一秒发送一个数据，共发射10次,结果输出 0~9
@Test
fun test() {
Observable.intervalRange(0, 10, 0, 1, TimeUnit.SECONDS)
                .subscribeOn(Schedulers.io())
                .observeOn(Schedulers.io())
                .subscribe(i -> System.out.println(i))
}
``` 
理所当然的认为，我们能看到输出台上输出0~9，事实上，数据还没有输出一个，测试代码就已经跑完了，所以结果是，输出台空空如也。
 
 单纯的使用Thread.sleep(10000)使当前的线程停留10秒不一定是最简单，但一定是最差的选择，因为这意味着我们需要花费更长时间去等待测试结果，更重要的问题是，并非每次我们都能直到异步操作的所需要的准确时间。

#### RxJavaPlugins
很快我找到了这篇文章：

[ Rxjava2单元测试的异步和同步转换](http://blog.csdn.net/verymrq/article/details/70833136)
 
 这篇文章是为数不多对我测试RxJava有用的文章之一，作者使用RxJavaPlugins，通过将RxJava所使用的线程换成Schedulers.trampoline()的方式，强制立即进行当前的任务，从而得到单元测试的结果。
 
代码如下：
```
    /**
     * 单元测试的时候，利用RxJavaPlugins将io线程转换为trampoline
     * trampoline应该是立即执行的意思（待商榷），替代了Rx1的immediate。
     */
    public static void asyncToSync() {
        RxJavaPlugins.reset();
        RxJavaPlugins.setIoSchedulerHandler(new Function<Scheduler, Scheduler>() {
            @Override
            public Scheduler apply(Scheduler scheduler) throws Exception {
                return Schedulers.trampoline();
            }
        });

```
然后再在测试类里加这么一句：
```
@Before
 public void setUp(){
     //将rx异步转同步
     RxjavaFactory.asyncToSync();
 } 
```
 令我振奋的是，我成功得到了预期的结果。

#### TestScheduler

我们通过上面的代码，解决了一大部分的异步测试需求，但是还有一个问题，那就是测试的时间，我们不希望这个测试花费这么长时间。

RxJava2的开发人员已经为我们考虑到了这个问题，对此他们提供了TestScheduler类，很明显，这也是一个Scheduler.

我们定义一个Rule，用来配置TestScheduler代替我们所使用的其他Scheduler，这是一个特殊的scheduler，它允许我们手动的将一个虚拟时间提前:

```kotlin
class RxSchedulerRule : TestRule {

    private var testScheduler: TestScheduler = TestScheduler()

    override fun apply(base: Statement?, description: Description?): Statement {
        return object : Statement() {
            @Throws(Throwable::class)
            override fun evaluate() {
                RxJavaPlugins.setIoSchedulerHandler { testScheduler }
                RxJavaPlugins.setComputationSchedulerHandler { testScheduler }
                RxJavaPlugins.setNewThreadSchedulerHandler { testScheduler }
                RxJavaPlugins.setSingleSchedulerHandler { testScheduler }
//                RxAndroidPlugins.setInitMainThreadSchedulerHandler { testScheduler }
                try {
                    base?.evaluate()
                } finally {
                    RxJavaPlugins.reset()
//                    RxAndroidPlugins.reset()
                }
            }
        }
    }
    
	//advanceTimeTo()以绝对的方式调整时间
    fun advanceTimeTo(delayTime: Long, timeUnit: TimeUnit) {
        testScheduler.advanceTimeTo(delayTime, timeUnit)
    }
    
	//advanceTimeBy()方法将调度器的时间调整为相对于当前位置的时间
    fun advanceTimeBy(delayTime: Long, timeUnit: TimeUnit) {
        testScheduler.advanceTimeBy(delayTime, timeUnit)
    }

    fun getScheduler(): TestScheduler {
        return testScheduler
    }
}
```
 对此，我们在每个测试类中添加这个Rule即可：

```kotlin
 @Rule
 @JvmField
 val rxRule = RxSchedulerRule()
```
> 在Kotlin的测试代码中，我们需要添加@JvmField注释，因为@Rule注释仅适用于字段和getter方法，但rxRule 是Kotlin中的一个属性。
 
 
 现在 我们再次进行测试：

```kotlin
class RxSchedulerRule_Test {

    @Rule
    @JvmField
    val rxRule = RxSchedulerRule()

    @Test
    fun testAdvanceTo_complete() {
        val observer = startRangeLoopThread()

        //将时间轴跳到第10秒，模拟执行发生的事件
        rxRule.advanceTimeTo(10, TimeUnit.SECONDS)

        assertComplete(observer)
    }

    private fun startRangeLoopThread(): TestObserver<Long> {
        val observer = TestObserver<Long>()
        val observable = getThread()
        observable.subscribe(observer)
        return observer
    }

    //开启一个子线程，每隔一秒发送一个数据，共发射10次,结果 0~9
    private fun getThread(): Observable<Long> {
        return Observable.intervalRange(0, 10, 0, 1, TimeUnit.SECONDS)
                .subscribeOn(Schedulers.io())
                .observeOn(Schedulers.newThread())
    }
	
	//断言
    private fun assertComplete(observer: TestObserver<Long>) {
        observer.assertValueCount(10)
                .assertComplete()
                .assertNoErrors()
                .assertValueAt(0, 0)
                .assertValueAt(9, 9)
    }
}
```
 这样我们仅仅需要数十或者数百毫秒，就能准确测试这个异步操作，并且验证我们预期的结果。
 

### 如何测试AndroidSchedulers.mainThread()

单纯的直接测试会导致报错：找不到AndroidSchedulers这个类。

我们将RxSchedulerRule 中的两行注解代码打开，这意味着当我们要使用AndroidSchedulers这个类时，RxAndroidPlugins已经用TestScheduler替换了AndroidSchedulers.mainThread()。

这样就解决了这个问题。
 
但是我很快遇到了一个奇怪的现象，那就是 

>当我一次运行整个测试类时，只有第一个测试通过，其余的测试通常会失败。
 
 这个现象困扰我很久，最终我找到的解决方式是，通过依赖注入解决问题：
 
参考 [【译】使用Kotlin和RxJava测试MVP架构的完整示例 - 第2部分 @ditclear](http://www.jianshu.com/p/0a845ae2ca64) 
 
 > 首先要做到这一点，我们需要创建一个SchedulerProvider接口，并提供两个实现。

>* AppSchedulerProvider - 这将为我们提供真正的调度器。 我们将把这个类注入所有的presenter，这将为我们的Rx订阅提供调度器。
* TestSchedulerProvide - 这个类将为我们提供一个TestScheduler而不是真正的scheduler。 当我们在测试中实例化我们的presenter时，我们将使用它作为它的构造函数参数。

```kotlin
interface SchedulerProvider {
    fun uiScheduler() : Scheduler
    fun ioScheduler() : Scheduler
}

class AppSchedulerProvider : SchedulerProvider {
    override fun ioScheduler() = Schedulers.io()
    override fun uiScheduler(): Scheduler = AndroidSchedulers.mainThread()
}

class TestSchedulerProvider() : SchedulerProvider {

    val testScheduler: TestScheduler = TestScheduler()

    override fun uiScheduler() = testScheduler
    override fun ioScheduler() = testScheduler
}
```
 通过依赖注入，在生产代码中使用AppSchedulerProvider ，在测试代码中使用TestSchedulerProvider。

以最初的MVP中的model和Presenter为例，测试代码如下：

* Model
```kotlin
class HomeModelTest : BaseTestModel() {

    private var retrofit: MockRetrofit = MockRetrofit()

    private var homeModel: HomeModel = HomeModel(mock())

    val provider = TestSchedulerProvider()

    @Before
    fun setUp() {
        homeModel.schedulers = provider
        homeModel = spy(homeModel)

        val service = retrofit.create(UserInfoService::class.java)
        whenever(homeModel.serviceManager.userInfoService).thenReturn(service)
    }

    @Test
    fun requestUserInfo() {
        val testObserver = TestObserver<UserInfo>()
        retrofit.path = MockAssest.USER_DATA
        doReturn(RxTestTransformer<UserInfo>()).whenever(homeModel).getUserInfoCache(anyString(), anyBoolean())

        val maybe = homeModel.requestUserInfo("qingmei2")
        maybe.subscribe(testObserver)

        provider.testScheduler.triggerActions()

        testObserver.assertValue { it -> it.login == "login" && it.name == "name" }
    }
}
```
 
* Presenter

```kotlin
class HomePresenterTest : BaseTestPresenter() {

    val view: HomeContract.View = mock()

    val model: HomeContract.Model = mock()

    var presenter: HomePresenter = HomePresenter(view, model)

    @Before
    fun setUp() {
        presenter = spy(presenter)
        doReturn(RxTestTransformer<Any>()).whenever(presenter).bindViewMaybe<Any>(view)
    }

    @Test
    fun requestUserInfo_success() {

        val s = MockAssest.readFile(MockAssest.USER_DATA)
        val user = Gson().fromJson(s, UserInfo::class.java)
        whenever(model.requestUserInfo(anyString())).thenReturn(Maybe.just(user))

        //testing
        presenter.requestUserInfo("qingmei2")

        val captor: ArgumentCaptor<UserInfo> = ArgumentCaptor.forClass(UserInfo::class.java)
        verify(view).onGetUserInfo(captor.capture());
        verify(view, never()).onError(anyString());

        assert(user.equals(captor.value))
    }

    @Test
    fun requestUserInfo_failed_error() {
        val s = MockAssest.readFile(MockAssest.error)
        val user = Gson().fromJson(s, UserInfo::class.java)
        whenever(model.requestUserInfo(anyString())).thenReturn(Maybe.just(user))

        presenter.requestUserInfo("qingmei2")

        verify(view, never()).onGetUserInfo(any());
        verify(view).onError("用户信息为空");
    }
}
```
 
## 小结

文笔所限，只是把思路整理一遍写了出来，很多类都没有详细去讲解，比如TestScheduler，比如TestObserver,这都是RxJava提供好的工具，我们一定要好好利用。

虽然思路有些天马行空，但是笔者最近数日整理出来的一套Kotlin的RxJava2+Retrofit的单元测试脚手架已经搭建完毕,并且作为笔者个人MVP架构下的测试框架，配以详细的单元测试代码，上传到了Github上，有需要的朋友可以参考：

[MvpArchitecture-Android](https://github.com/qingmei2/MvpArchitecture-Android)
 

## 参考文章

测试RxJava2
http://www.infoq.com/cn/articles/Testing-RxJava2

【译】使用Kotlin和RxJava测试MVP架构的完整示例 - 第2部分：
http://www.jianshu.com/p/0a845ae2ca64
 
 
 
 
 
 
 
