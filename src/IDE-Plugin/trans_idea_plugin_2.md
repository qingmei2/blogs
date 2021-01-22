# [译] 编写AndroidStudio插件(二):持久化数据

> 原文：[Write an Android Studio Plugin Part 2: Persisting data](https://proandroiddev.com/write-an-android-studio-plugin-part-2-persisting-data-5f81ca5d9e43)   
> 作者：[Marcos Holgado](https://medium.com/@marcosholgado)   
> 译者：[却把清梅嗅](https://github.com/qingmei2)   
>《编写AndroidStudio插件》系列是 IntelliJ IDEA 官方推荐的学习IDE插件开发的博客专栏，希望对有需要的读者有所帮助。

在本系列的[第一部分](https://proandroiddev.com/write-an-android-studio-plugin-part-1-creating-a-basic-plugin-af956c4f8b50)中，我们了解了如何为`Android Studio`创建一个基本的插件，并编写了第一个`Action`。本文我们将了解如何在插件中对数据进行持久化。

![image](https://raw.githubusercontent.com/qingmei2/qingmei2-blogs-art/master/ide-plugin/image.howuoqg73xb.png)

请记住，您可以在`GitHub`上找到本系列的所有代码，还可以在对应的分支上查看每篇文章的相关代码，本文的代码在`Part2`分支中。

> https://github.com/marcosholgado/plugin-medium

## 我们要做什么？

我们今天的目标是在插件中对数据进行持久化。在此过程中，我们将学习什么是`component`以及如何使用`component`来管理插件的生命周期。接下来我们将开始使用它，以保证`Android Studio`启动并且安装了新版本的插件时显示通知。您将来可用此功能向用户显示您插件更新了哪些内容。

## 什么是 Component ？

在编写任何代码之前，我们需要了解什么是`Component`。 `Component`是插件集成的基本概念，`Component`让我们能够控制插件的生命周期并保持其状态，以便将其自动保存和加载。

一共有三种不同类型的`Component`：

* **Application** 级别: 在`IDE（Android Studio）`启动时创建并初始化的`Component`;
* **Project** 级别: 为每个项目实例创建的`Component`;
* **Module** 级别: 为每个项目中的模块创建的`Compoenent`.

## 第一步：新建一个 Component

首先我们要确定所需`Component`的类型，在本文中，我们想在`Android Studio`启动时做一些事情，因此，通过查看不同类型的`Component`，我们可以清楚地知道需要一个 **`Application`级别的`Component`**。

`JetBrains`官方文档这样描述：我们可以实现`ApplicationComponent`接口，这是可选的，但本文我们会这样做。`ApplicationComponent`接口将为我们提供之前提到的生命周期方法，便于我们在`IDE`启动时用于执行某些操作。这些方法来自`ApplicationComponent`的扩展类`BaseComponent`。

```java
public interface BaseComponent extends NamedComponent {
  /**
   * Component should perform initialization and communication with other components in this method.
   * This is called after {@link com.intellij.openapi.components.PersistentStateComponent#loadState(Object)}.
   */
  default void initComponent() {
  }

  /**
   * @see com.intellij.openapi.Disposable
   */
  default void disposeComponent() {
  }
}
```

现在让我们对`Component`进行编码，第一次迭代将非常简单，我们通过继承`ApplicationComponent`并重写`initComponent`来检查是否有新版本。

```kotlin
class MyComponent: ApplicationComponent {

    override fun initComponent() {
        super.initComponent()
        if (isANewVersion()) { }
    }

    private fun isANewVersion(): Boolean = true

}
```

## 第二步：实现 isANewVersion()

那么有意思的来了。我们将从声明两个新字段开始：`localVersion`和`version`。第一个将存储我们已安装的最新版本，而第二个将是我们插件的实际安装版本。

我想要比较它们，以检查版本号`localVersion`是否在`version`的后面，若如此我们就能知道用户是否刚刚安装了新版本的插件，以及是否向用户发送通知。我们还必须将`localVersion`更新为与`version`相同的值，以便下次用户启动`Android Studio`时不再收到欢迎信息。

首先，用户未安装我们的插件，因此我们将`localVersion`的值设置为`0.0`，因为我们的第一个版本是`1.0-SNAPSHOT`。您可以在`build.gradle`文件中更改插件的版本，但如果未更改，则应为：

```groovy
version '1.0-SNAPSHOT'
```

为了实现`isANewVersion`，我们将做一些简单的事情。如果你愿意，可以适当进行修改，但是这里只是针对版本号进行简单的比较，因此我没有对边界条件相关逻辑进行全方位覆盖。

首先，我摆脱了实际上要在版本上维护的`-SNAPSHOT`部分。之后，我使用`.`作为分隔符对每个版本号进行了分割。因此我们得到一个包含所有主数字和副数字的`List<String>`，以便我们可以比较它们并返回`true`或`false`。这并非最佳算法，但足以满足我们的需求。该算法的假设非常简单，两个版本号必须具有相同的长度，并且必须遵循相同的命名约定`majorV.minorV`。

```kotlin
private fun isANewVersion(): Boolean {
    val s1 = localVersion.split("-")[0].split(".")
    val s2 = version.split("-")[0].split(".")

    if (s1.size != s2.size) return false
    var i = 0

    do {
        if (s1[i] < s2[i]) return true
        i++
    } while (i < s1.size && i < s2.size)

    return false
}
```

## 第三步：整合起来

现在是时候将`isANewVersion`方法以及两个新字段移到`Component`中了。您可以使用`PluginManager`获得插件的版本以及插件的`ID`，您还可以在`plugin.xml`文件中找到（和更改）插件的`ID`。我还建议此时将插件名称更改为更有意义的名称，例如`My awesome plugin`。

```xml
<idea-plugin>
    <id>myplugin.myplugin</id>
    <name>My awesome plugin</name>
    ...
```

该`Component`的整个实现非常简单:我们从`PluginManager`获取版本，检查是否发生了版本更新，如果有，就同步更新本地版本号，以便下次启动`Android Studio`时不会再触发整个过程。最后，我们向用户显示一个简单的通知，整合完毕后，看起来像这样。

```kotlin
class MyComponent: ApplicationComponent {

    private var localVersion: String = "0.0"
    private lateinit var version: String

    override fun initComponent() {
        super.initComponent()

        version = PluginManager.getPlugin(
            PluginId.getId("myplugin.myplugin")
        )!!.version

        if (isANewVersion()) {
            updateLocalVersion()
            val noti = NotificationGroup("myplugin",
                                         NotificationDisplayType.BALLOON,
                                         true)

            noti.createNotification("Plugin updated",
                                    "Welcome to the new version",
                                   NotificationType.INFORMATION,
                                   null)
                .notify(null)
        }
    }

    private fun isANewVersion(): Boolean {
        val s1 = localVersion.split("-")[0].split(".")
        val s2 = version.split("-")[0].split(".")

        if (s1.size != s2.size) return false
        var i = 0

        do {
            val l1 = s1[i]
            val l2 = s2[i]
            if (l1 < l2) return true
            i++
        } while (i < s1.size && i < s2.size)

        return false
    }

    private fun updateLocalVersion() {
        localVersion = version
    }

}
```

## 第四步：持久化 Component 的状态

现在，我们可以尝试测试我们的插件，由于两个原因，它无法正常工作。

首先是因为我们仍然必须注册`Component`，其次是因为我们还没有真正保留`Component`的状态。每次`Android Studio`初始化时，`localVersion`的值仍然是`0.0`。

那么，针对如何保存`Component`状态的问题，我们该怎么做呢？为此，我们必须实现`PersistentStateComponent`。这意味着我们必须重写两个新方法，即`getState`和`loadState`。我们还需要了解`PersistentStateComponent`的工作方式，
它将公共字段，带注释的私有字段和`bean`属性存储为`XML`格式。要从序列化中删除公共字段，可以使用`@Transient`注释该字段。

在我们的例子中，我将使用`@Attribute`注释`localVersion`，以便在存储它的同时将其保持私有状态，将组件本身返回到`getState`并使用`XmlSerializerUtil`加载状态，我们还必须使用`@State`注释组件，并指定`xml`的位置。

```kotlin
@State(
        name = "MyConfiguration",
        storages = [Storage(value = "myConfiguration.xml")])
class MyComponent: ApplicationComponent,
        PersistentStateComponent<MyComponent> {

    @Attribute
    private var localVersion: String = "0.0"
    private lateinit var version: String

    override fun initComponent() {
        super.initComponent()

        version = PluginManager.getPlugin(
                PluginId.getId("myplugin.myplugin")
        )!!.version

        if (isANewVersion()) {
            updateLocalVersion()
            val noti = NotificationGroup("myplugin",
                                         NotificationDisplayType.BALLOON,
                                         true)
            noti.createNotification("Plugin updated",
                                    "Welcome to the new version",
                                    NotificationType.INFORMATION,
                                    null)
                .notify(null)
        }
    }

    override fun getState(): MyComponent? = this

    override fun loadState(state: MyComponent) = XmlSerializerUtil.copyBean(state, this)

    private fun isANewVersion(): Boolean {
        val s1 = localVersion.split("-")[0].split(".")
        val s2 = version.split("-")[0].split(".")

        if (s1.size != s2.size) return false
        var i = 0

        do {
            if (s1[i] < s2[i]) return true
            i++
        } while (i < s1.size && i < s2.size)

        return false
    }

    private fun updateLocalVersion() {
        localVersion = version
    }

}
```

像往常一样，您可以按需定制，例如定义存储位置或自定义`xml`格式，更多信息请参考[这里](https://www.jetbrains.org/intellij/sdk/docs/basics/persisting_state_of_components.html)。

## 第五步：注册 Component

最后一步是在`plugin.xml`文件中注册`Component`。为此，我们只需要在文件的`application-component`部分内创建一个新`Component`，然后指定我们刚刚创建的`Component`类即可。

```xml
<application-components>
    <component>
        <implementation-class>
            components.MyComponent
        </implementation-class>
    </component>
</application-components>
```

大功告成！现在，您可以运行`buildPlugin`命令，使用`Android Studio`中生成的`jar`文件从磁盘安装插件，下次打开`Android Studio`时，您会看到此通知。之后，仅当您提高插件版本时，通知也会再次出现。

![image](https://raw.githubusercontent.com/qingmei2/qingmei2-blogs-art/master/ide-plugin/image.qy3u8uk92i.png)

第2部分到此为止，在[第3部分](https://proandroiddev.com/write-an-android-studio-plugin-part-3-settings-662a535c6962)中，我们将学习如何为插件创建设置页面，当然在此之前我们还将了解如何为插件创建`UI`。

如果您有任何疑问，请访问[Twitter](https://www.twitter.com/orbycius)或发表评论。

---
## 《编写AndroidStudio插件》译文系列

* [译: 编写AndroidStudio插件(一):创建一个基本插件](https://github.com/qingmei2/blogs/issues/50)
* [译: 编写AndroidStudio插件(二):持久化数据](https://github.com/qingmei2/blogs/issues/51)
* [译: 编写AndroidStudio插件(三): 更多配置](https://github.com/qingmei2/blogs/issues/52)
* [译: 编写AndroidStudio插件(四):整合Jira](https://github.com/qingmei2/blogs/issues/53)
* [译: 编写AndroidStudio插件(五):本地化和通知](https://github.com/qingmei2/blogs/issues/54)

## 关于译者

Hello，我是 [却把清梅嗅](https://github.com/qingmei2) ，如果您觉得文章对您有价值，欢迎 ❤️，也欢迎关注我的 [博客](https://blog.csdn.net/mq2553299) 或者 [GitHub](https://github.com/qingmei2)。

如果您觉得文章还差了那么点东西，也请通过 **关注** 督促我写出更好的文章——万一哪天我进步了呢？

* [我的Android学习体系](https://github.com/qingmei2/blogs)
* [关于文章纠错](https://github.com/qingmei2/blogs/blob/main/error_collection.md)
* [关于知识付费](https://github.com/qingmei2/blogs/blob/main/appreciation.md)
* [关于《反思》系列](https://github.com/qingmei2/blogs/blob/main/src/%E5%8F%8D%E6%80%9D%E7%B3%BB%E5%88%97/thinking_in_android_index.md)
