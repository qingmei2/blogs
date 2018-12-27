> 如果您不是很了解Android的自动化测试，或者还不了解**UI自动化测试对于Android开发者的意义**，请参考玉刚说的文章**[《解放双手，Android开发应该尝试的UI自动化测试》](https://mp.weixin.qq.com/s/ODbqUHjQUTA79UyI5Fw5Mw)**。

## 概述
* * *

> 本文已授权 微信公众号 **玉刚说** （[@任玉刚](https://blog.csdn.net/singwhatiwanna/)）独家发布。

我是**[却把清梅嗅](https://blog.csdn.net/mq2553299)**，一个普通的Android开发者，除了日常工作之外，我还喜欢在[我的Github](https://github.com/qingmei2)上开源分享自己写的一些小工具。其中我个人比较满意的是[RxImagePicker](https://github.com/qingmei2/RxImagePicker),它是我花费业余时间实现的一个Android的**响应式图片选择器**，它的项目主页：

 > https://github.com/qingmei2/RxImagePicker

随着一些小伙伴的支持，这个图片选择器慢慢被尝试应用在了一些项目中，随着用的人越来越多，我感到**压力越来越大**，至少我需要保证，每次发布的新版本要**避免低级的失误**，至少不能发生**一使用就崩溃**的情况吧。

我很快意识到我遇到了**困境**——即使是库的一个小版本的更新，我都需要保证库中每个界面基本功能的可用，版本发布后，我都需要自己去依赖Jcenter上最新的版本，然后运行并**手动测试**它的各个界面。

就这样，我坚持了几个版本的迭代，迭着迭着，我就迭不动了。

![](https://upload-images.jianshu.io/upload_images/7293029-22ac649a788f42cd.jpeg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

我不知道还能坚持**手动划来划去测试**多久，这意味着，**UI的自动化测试势在必行**，于是我借着这个机会去做了。结果是：**UI的自动化测试被应用到了我的这个项目中**。

现在，每次发布版本，我只需要一键运行，**避免了不会产生低级bug同时，免去了每个界面手动测试的繁琐**：

![UI自动化测试效果](https://upload-images.jianshu.io/upload_images/7293029-93858b014e76c088.gif?imageMogr2/auto-orient/strip)


我需要做的就是一遍惬意喝茶，一遍等待自动化测试的结果，很快，我得到了下面的结果：

![](https://upload-images.jianshu.io/upload_images/7293029-3af80b72defac98b.jpeg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

测试代码其实并不多，但是也的确花费了我不少的闲暇时间去学习Espresso的UI自动化测试，但我认为这是值得的。

当然，因为Android的UI自动化测试在国内并未广泛应用开来（实际上不只是UI自动化测试，单元测试也是如此，我个人猜想，和国内普遍性的焦躁心态不无关系），这方便的学习资料很少，我也多多少少踩了一些坑，我决定将我的实践经历分享出来，希望对一些想要学习Espresso的朋友有一定的帮助。

## 准备

* * *

本文默认读者对Android**UI自动化测试的基本概念**有一定的了解，并且**初步掌握了Espresso**工具库的使用。

如果您还不是很熟悉这些基础的API，请参考笔者的这篇文章：

> **[《解放双手，Android开发应该尝试的UI自动化测试》](https://mp.weixin.qq.com/s/ODbqUHjQUTA79UyI5Fw5Mw)**

此外，如果读者有一定的**测试相关的基础**，包括**JUnit4，Rule**的概念，以及**Kotlin**的基本语法就更好了。

本文示例的所有**测试代码**都源自此项目：

> https://github.com/qingmei2/RxImagePicker

如果觉得这个库还不错，也欢迎star或者fork它，这也算是笔者写本文时的一个私心吧(抱拳)...

## 步步为营，实践中遇到的问题

* * *

### 1.依赖和配置

首先是在自己项目的build.gradle文件中**添加Espresso的相关依赖**以及**配置AndroidJUnitRunner**：

```groovy
android {
    defaultConfig {
        // ...
        testInstrumentationRunner 'android.support.test.runner.AndroidJUnitRunner'
    }
}
dependencies {

  //...

  androidTestImplementation "com.android.support.test.espresso:espresso-core:3.0.2"
  androidTestImplementation "com.android.support.test.espresso:espresso-contrib:3.0.2"
  androidTestImplementation "com.android.support.test.espresso:espresso-idling-resource:3.0.2"
  androidTestImplementation "com.android.support.test.espresso:espresso-intents:3.0.2"
  androidTestImplementation 'com.android.support.test:runner:1.0.2'
  androidTestImplementation 'com.android.support.test:rules:1.0.2'
}
```

> 另：Groovy是一个非常好用的语言，在我的个人项目中，每次我发布新的版本，我需要让sample去依赖Jcenter远端的版本；而开发时会去依赖project中的Module，这样通过添加配置一个变量，作为**开关**进行**版本控制**即可：

![](https://upload-images.jianshu.io/upload_images/7293029-c75988e6279da323.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

写好后，就可以在Module下的androidTest包下开始自己的**UI自动化测试**了：
![](https://upload-images.jianshu.io/upload_images/7293029-e10ec91fa1610af2.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

首先我们从简单的开始，sample的主页面：

### 2.测试界面跳转（Intent）

![](https://upload-images.jianshu.io/upload_images/7293029-75ef499b858507af.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

主页面非常简单，3个按钮，跳转3个不同的图片选择界面，对于单一的按钮测试代码如下：

```Kotlin
@RunWith(AndroidJUnit4::class)
@LargeTest
class MainActivityTest {

    val TEST_PACKAGE_NAME = "com.qingmei2.sample"

    @Rule
    @JvmField
    var tasksActivityTestRule = IntentsTestRule<MainActivity>(MainActivity::class.java)

    @Test
    fun testJump2SystemActivity() {
        // 检查点击事件——点击SystemTheme按钮，跳转系统选择器界面
        checkScreenJumpEvent({ R.id.btn_system_picker },
                { ".system.SystemActivity" })
    }

    private fun checkScreenJumpEvent(buttonId: () -> Int,
                             shortName: () -> String,
                             packageName: () -> String = { TEST_PACKAGE_NAME }) {

        // 点击对应按钮
        onView(withId(buttonId())).perform(click()).check(doesNotExist())
        // 是否有对应的intent产生
        intending(allOf(
                toPackage(packageName()),    //  包路径
                hasComponent(hasShortClassName(shortName()))  //类的shortClassName
        ))

        // 点击返回键，检查是否回到当前界面
        pressBack()
        onView(withId(buttonId())).check(matches(isDisplayed()))
    }
}
```
对于页面的跳转测试，Espresso提供了**IntentsTestRule**以代替**ActivityTestRule**，同时它提供了**对界面元素发生的Intent跳转行为的检查机制**。

因此，我们对于**界面跳转的检测**，只需要将**ActivityTestRule**替换为**IntentsTestRule**即可，不需要多余的配置。

当然，导致这种简便性的真正原因是，**IntentsTestRule**本身就是继承了**ActivityTestRule**：

```Java
public class IntentsTestRule<T extends Activity> extends ActivityTestRule<T> {}
```

### 3.测试权限请求

接下来的图片选择界面的测试，以微信主题为例，界面如下：

![](https://upload-images.jianshu.io/upload_images/7293029-b906fbbcbfbcb96d.gif?imageMogr2/auto-orient/strip)

正如我自己操作的，我分别需要添加模拟用户**打开相机**和用户**打开相册**的两种行为的测试代码。

其实这个测试并不难写，以打开微信主题的相册界面为例，测试代码如下：

```kotlin
@RunWith(AndroidJUnit4::class)
@LargeTest
class WechatActivityTest {
    // 打开相册当然是需要调用startActivityForResult获取结果
    // 因此这里Mock成功后的activityResult（根据实际项目中的参数来）
    private val successActivityResult: Instrumentation.ActivityResult =
            with(Intent()) {
                putExtra(BasePreviewActivity.EXTRA_RESULT_BUNDLE, EXTRA_BUNDLE)
                putExtra(BasePreviewActivity.EXTRA_RESULT_APPLY, EXTRA_RESULT_APPLY)

                Instrumentation.ActivityResult(Activity.RESULT_OK, this)
            }

    @Rule
    @JvmField
    var systemActivityTestRule = IntentsTestRule<WechatActivity>(WechatActivity::class.java)

    @Test
    fun testPickGallery() {
        intending(allOf(
                toPackage("com.qingmei2.rximagepicker_extension_wechat"),
                hasComponent(".ui.WechatImagePickerActivity")
        )).respondWith(successActivityResult)

        onView(withId(R.id.imageView)).check(matches(isDisplayed()))

        onView(withId(R.id.fabGallery)).perform(click())

        onView(withId(R.id.imageView)).check(doesNotExist())
    }

    companion object Mock {
        private const val EXTRA_BUNDLE = "123"
        private const val EXTRA_RESULT_APPLY = "456"
    }
}
```

这里看起来没有什么问题，我们直接运行，Espresso的代码模拟按钮的点击事件，结果并没有出现想象中的界面跳转，原因就是——

> 在进入相册界面之前，系统弹出了一个权限请求的弹窗。

我的UI界面逻辑是，如果用户没有赋予权限，那么不会跳转接下来的界面，而权限弹窗是**系统级**的，我们无法通过Espresso找到对应的Button进行权限的确认。

依赖于求助Google，我发现所幸在比较新的版本中，Espresso提供了GrantPermissionRule，它可以自动赋予界面所弹出权限弹窗对应的权限：

```kotlin
@Rule
@JvmField
var grantPermissionRule = GrantPermissionRule.grant(
      android.Manifest.permission.WRITE_EXTERNAL_STORAGE
)
```
我在测试类中添加了WRITE_EXTERNAL_STORAGE的权限Rule，果然在接下来的测试中，没有再出现因为权限弹窗导致出现的测试失败情况。

### 4.Application & Library ？

接下来我要讲述的这个问题困扰了我很久，在讲述它之前，我先放张图：

![](https://upload-images.jianshu.io/upload_images/7293029-071428ab7944931c.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

正如您所见的，这是项目的结构，sample作为工具库的上层调用者，底层不同主题的图片选择UI界面被放在了library module中（比如上文所展示的微信主题界面WechatImagePickerActivity）。

我认为WechatImagePickerActivity的UI测试也应该放在library下——这似乎理所当然，但当我写好相关的测试代码后，在运行时，我发现一个问题，**相册界面没有显示任何图片**，就好像手机里没有任何照片一样。

我苦思冥想很久，最终找到了问题的关键，我发现，当我把该Activity的UI测试代码放在library包下，那么我的自定义相册界面**无法找到任何图片资源**；而如果我把该Activity的UI测试代码放在application包下，那么我的自定义相册界面就**能够正常显示了**。

我并不知道发生这种情况的真正原因，但是找到了这个原因已经足够，我把所有的UI测试代码都暂时放在了sample的androidTest目录下了。

### 5.测试UI前使用依赖注入

**并非所有界面都可以直接测试**，一些Activity在启动的同时是需要一些额外依赖的，举例来说，我的项目中，微信主题界面的WechatImagePickerActivity的onCreate()中，需要一个这样的对象：

```kotlin
class SelectionSpec private constructor() : ICustomPickerConfiguration {
    //....各种各样的配置，比如最大可选数量，themeId等
    var themeId: Int = 0
    var orientation: Int = 0
    var countable: Boolean = false
    var maxSelectable: Int = 0
    var maxImageSelectable: Int = 0
    var maxVideoSelectable: Int = 0
    var filters: ArrayList<Filter>? = null
    var capture: Boolean = false
    var captureStrategy: CaptureStrategy? = null
    var spanCount: Int = 0
    var gridExpectedSize: Int = 0
    var thumbnailScale: Float = 0.toFloat()

    // Builder 不再显示
}

class WechatImagePickerActivity : AppCompatActivity() {
    // 在Activity onCreate()中，需要SelectionSpec的对象，若为空，则会crash
    override fun onCreate(savedInstanceState: Bundle?) {
        setTheme(SelectionSpec.instance!!.themeId)
        super.onCreate(savedInstanceState)
        setContentView(R.layout.activity_picker_wechat)
    }
    // ...
}
```

由此可见，有些情况下，如果不提前为界面添加必须的依赖，UI测试是根本无法进行的。

这些对象可能会在API的调用过程中，库的内部进行了解析以及初始化，但是对于单个UI界面的测试来讲，这些对象却并不一定会被初始化。正如你所见的，本文中的Activity就在这种情况下抛出NullPointException。

依赖的实例化取决于项目或者库的架构设计，完成之后，只需要通过ActivityTestRule提供的`beforeActivityLaunched()`进行依赖的注入即可,这个方法会在Activity启动之前执行：

```kotlin
    @Rule
    @JvmField
    val tasksActivityTestRule =
            object : IntentsTestRule<WechatImagePickerActivity>(WechatImagePickerActivity::class.java) {

                override fun beforeActivityLaunched() {
                    super.beforeActivityLaunched()

                    // Inject the ICustomPickerConfiguration
                    SelectionSpec.instance = WechatConfigrationBuilder(MimeType.ofImage(), false)
                            .maxSelectable(9)
                            .countable(true)
                            .spanCount(3)
                            .build()
                }
            }
```

现在我们在Activity启动前配置好了所需要的依赖，运行测试代码，NullPointException也不再发生了。

### 6.对RecyclerView的操作进行测试

Espresso在新的版本（好像是2.2+）中添加了`RecyclerViewAction`，以对应我们想要对RecyclerView的操作，这极大方便了开发者进行使用（要知道早期版本中是没有对RecyclerView的支持，这样我们想对item进行操作，就必须依赖`onData()`）。

`RecyclerViewAction`很强大，但我不会针对它的如何使用进行过多的讲解，以一个简单的小例子进行阐述，当我们想操作一个item中的某个指定的View进行操作，我们可以通过自定义ViewAction来实现：

```kotlin
fun clickRecyclerChildWithId(id: Int): ViewAction =
        object : ViewAction {
            override fun getDescription(): String =
                    "Click on a child view with specified id."

            override fun getConstraints(): Matcher<View>? =
                    null

            override fun perform(uiController: UiController, view: View) {
                view.findViewById<View>(id).apply {
                    performClick()
                }
            }
        }

fun ViewInteraction.clickRecyclerChildWithId(itemPosition: Int,
                                             viewId: Int) =
        perform(RecyclerViewActions.actionOnItemAtPosition<RecyclerView.ViewHolder>(
                itemPosition, clickRecyclerChildWithId(viewId)
        ))
```

以个人项目为例，微信相册界面，我想点击指定Position Item的CheckView以选中某张图片，测试代码就可以这样:

```kotlin
// select image
onView(withId(R.id.recyclerview))
    .perform(actionOnItemAtPosition<RecyclerView.ViewHolder>(
                1, clickRecyclerChildWithId(R.id.check_view)  //顶层函数
    ))
```

`Kotlin`的**顶层函数**非常实用，加上**拓展函数**，会使得上述的测试代码变得更加简洁：

```kotlin
// 点击poisition = 1 的item的CheckView
onView(withId(R.id.recyclerview))
        .clickRecyclerChildWithId(1, R.id.check_view)   //拓展函数
```

### 7.测试Activity是否已经Finish

除了系统的Back键，很多界面都有**返回功能的按钮设计**，甚至其他界面元素，它们会导致当前Activity的关闭，这个该如何测试呢？

我翻遍了Espresso的官方文档，都没有找到对Activity是否关闭的CheckAPI，并且，我诧异的发现，无论是百度还是google，我都没找到关于这个情况的讨论。

我一时间束手无策，我不认为这种测试Case没有工程师想到过，他们是如何处理的呢？

我换了一个思考的角度，为什么Espresso没有提供这样的API给开发者——除非是，**API本身就已经存在了**。

我最终的解决方案是借助于`ActivityTestRule`和`JUnit4`。

`ActivityTestRule`本身就可以提供正在测试中的Activity:

```kotlin
fun ActivityTestRule<out Activity>.isFinished(): Boolean = activity.isFinishing
```

这之后，配合`JUnit4`本身的断言完全可以实现**Activity是否已经关闭的校验**，我只需要这样调用：

```kotlin
Assert.assertTrue(activityTestRule.isFinished())
```

这么简单的实现方式让我感到哭笑不得，所幸虽然耽误了一些时间，但我得到了我想要的结果。

## 小结

* * *

虽然经历了各种各样奇怪的问题（踩坑），好在有惊无险，成功上岸，关于Android的测试相关文献（代码demo）一向甚少，希望本文能够为正在学习UI自动化测试的同行们提供一些可行性的建议和指导。

本文项目地址：

> https://github.com/qingmei2/RxImagePicker

也希望本文能够让一些朋友体会到UI自动化测试的好处，即使覆盖实现的过程非常曲折，但是当真正实现之后，才会真正爱上它，正所谓：

> 金风玉露一相逢，便胜却人间无数。


![](https://upload-images.jianshu.io/upload_images/7293029-93858b014e76c088.gif?imageMogr2/auto-orient/strip)
