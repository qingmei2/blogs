## 简述

在日常项目开发中，基本没有什么机会用到Kotlin，几个月前学习的语法，基本上都忘光了，于是自己强迫自己在写Demo中使用Kotlin，同时，在目前开发的项目中开了一个测试分支，用来补全之前没有写的测试代码。

## 环境配置

### 1.MockAPI

单元测试中使用真实开发环境中的真实数据是不明智的，最好的方式是用本地的数据模拟网络请求，比如说我们有这样一个API，联网library我们选择Retrofit:

```
//TestService
interface TestService {

    @GET("/test/api")
    abstract fun getUser(@Query("login") login: String): Observable<User>
}
```

我们本地mock这个API会返回这样的Json数据：
```
{
    "login": "qingmei2",
    "name": "qingmei"
}
```

对应的data类：
```
class User(val name: String = "defaultName",
           val login: String = "defaultLogin")
```

好的，接下来我们定义一个Asset类，负责管理本地Mock的API返回资源：

```
object MockAssest {

    private val BASE_PATH = "app/src/test/java/cn/com/xxx/xxx/base/mocks/data"
    
    //User API对应的模拟json数据的文件路径
    val USER_DATA = BASE_PATH + "/userJson_test"
	
	//通过文件路径，读取Json数据
    fun readFile(path: String): String {
        val content = file2String(File(path))
        return content
    }
	//kotlin丰富的I/O API,我们可以通过file.readText（charset）直接获取结果
    fun file2String(f: File, charset: String = "UTF-8"): String {
        return f.readText(Charsets.UTF_8)
    }
}
```

关于Kotlin更多强大的IO操作的API，可以参考这篇：[Kotlin IO操作](http://www.jianshu.com/p/adaf5136e197)

### 2.MockRetrofit

我们直接配置一个MockRetrofit进行API的拦截：

```
class MockRetrofit {

    var path: String = ""

    fun <T> create(clazz: Class<T>): T {

        val client = OkHttpClient.Builder()
                .addInterceptor(Interceptor { chain ->
                    val content = MockAssest.readFile(path)
                    val body = ResponseBody.create(MediaType.parse("application/x-www-form-urlencoded"), content)
                    val response = Response.Builder()
                            .request(chain.request())
                            .protocol(Protocol.HTTP_1_1)
                            .code(200)
                            .body(body)
                            .message("Test Message")
                            .build()
                    response
                }).build()

        val retrofit = Retrofit.Builder()
                .baseUrl("http://api.***.com")
                .client(client)
                .addCallAdapterFactory(RxJava2CallAdapterFactory.create())
                .addConverterFactory(GsonConverterFactory.create())
                .build()
        return retrofit.create(clazz)
    }
}
```

这样我们直接通过MockRetrofit.create(APIService.class)直接mock一个对应的API Service对象。

### 3.对上面两个tool类的测试

在测试自己的业务代码之前，我们当然要先保证这两个工具类的逻辑正确，如果这两个脚手架都是错误的，那么接下来业务代码的单元测试毫无意义。

* MockAsset.kt的Test

```
class MockAssetTest {

    @Test
    fun assetTest() {
	    //MockAssest读取文件，该函数所得结果将来会作为模拟的网络数据返回，我们这个单元测试的意义
	    //就是保证模拟的网络数据能够正确的返回
        val content = MockAssest.readFile(MockAssest.USER_DATA)
        Observable.just(content)
                .test()
                .assertValue("{\n" + "    \"login\": \"qingmei2\",\n" + "    \"name\": \"qingmei\"\n" + "}")
    }
}
```
*  MockRetrofit.kt的Test

```
class MockRetrofitTest {
	
    @Test
    fun mockRetrofitTest() {
	    // 这个测试是保证Retrofit能够成功拦截API请求，并返回本地的Mock数据
        val retrofit = MockRetrofit()
        val service = retrofit.create(TestService::class.java)
        retrofit.path = MockAssest.USER_DATA  //设置Path，设置后，retrofit会拦截API,并返回对应Path下Json文件的数据

        service.getUser("test")
                .test()
                .assertValue { it ->
                    it.login.equals("qingmei2")
                    it.name.equals("qingmei")
                }
    }
}
```

## 使用 Mockito-Kotlin

我尝试使用这个新的库，[mockito-kotlin：Using Mockito with Kotlin](https://github.com/nhaarman/mockito-kotlin)

我选择这个基于Mockito之上的拓展库的理由很简单，更方便入门（我无法保证之后的测试代码过程中会不会踩坑，但是首先我得能够进行单元测试）。

关于Mockito在Kotlin的使用中会遇到的一些问题，这篇文章也许会对你有些帮助：

[在Kotlin上怎样用Mockito2 mock final 类（KAD 23)](http://www.cnblogs.com/figozhg/archive/2017/05/06/6817848.html)

我没有按照上面的步骤进行配置的尝试，但是当我在使用[mockito-kotlin](https://github.com/nhaarman/mockito-kotlin)踩到坑时，在这篇笔记中留下这样一个后路，也许不会让我碰得头破血流而束手无策。

### Mock依赖

我的项目中使用MVVM的架构，这意味着，ViewModel的测试至关重要。

首先我把一些常用的依赖放到了BaseViewModel中：

```
public class BaseViewModel {

    @Inject
    protected AccountManager accountManager;//账户相关
    @Inject
    protected ServiceManager serviceManager;//API相关
    
	//保存不同的加载状态
    public final ObservableField<State> loadingState = new ObservableField<>(LOAD_WAIT);
	...
	...
	...
}
``` 

我写了一个BaseTestViewModel类，他继承了BaseViewModel，这意味着同样持有accountManager和serviceManager。

我在setUp函数中初始化了这两个重要的对象，并进行简单的测试：

```
open class BaseTestViewModel : BaseViewModel() {

    @Before
    fun setUp() {
        accountManager = mock()
        serviceManager = mock()
    }
	
	//测试accountManager 成功Mock
    @Test
    fun testAccountManager() {
        Assert.assertNotNull(accountManager)
        whenever(accountManager.toString()).thenReturn("mock AccountManager.ToString!")
        Assert.assertEquals(accountManager.toString(), "mock AccountManager.ToString!")
    }
    
	//测试serviceManager 成功Mock
    @Test
    fun testServiceManager() {
        Assert.assertNotNull(serviceManager)
        val alertService = mock<AlertService>()
        whenever(alertService.toString()).thenReturn("mock alertService")
        whenever(serviceManager.alertService).thenReturn(alertService)
        Assert.assertEquals(serviceManager.alertService.toString(), "mock alertService")
    }

    class TestViewModel : BaseTestViewModel() {
		//测试BaseTestViewModel的子类也能成功持有mock好了的accountManager
        @Test
        fun testSubTestClass() {
            Assert.assertNotNull(accountManager)
            whenever(accountManager.toString()).thenReturn("mock AccountManager.Sub ToString!")
            Assert.assertEquals(accountManager.toString(), "mock AccountManager.Sub ToString!")
        }
    }
}
```

这几个测试pass之后，我可以尝试对我的不同业务代码下的ViewModel进行测试了。

## 交流

本文是简单的尝试下搭建的测试脚手架，如果您有更好的方式或思路，望请不吝指出。
