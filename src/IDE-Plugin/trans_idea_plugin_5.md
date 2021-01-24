# [译] 编写AndroidStudio插件(五):本地化和通知

> 原文：[Write an Android Studio Plugin Part 5: Localization and Notifications](https://proandroiddev.com/write-an-android-studio-plugin-part-5-localization-and-notifications-cb036d867587)   
> 作者：[Marcos Holgado](https://medium.com/@marcosholgado)   
> 译者：[却把清梅嗅](https://github.com/qingmei2)   
>《编写AndroidStudio插件》系列是 IntelliJ IDEA 官方推荐的学习IDE插件开发的博客专栏，希望对有需要的读者有所帮助。

在本系列的[第四部分](https://proandroiddev.com/write-an-android-studio-plugin-part-4-jira-integration-cd54df01cff6)中，我们学习了如何在插件中集成诸如`Jira Cloud Platform`之类的第三方`API`，以及如何使用`MVP`或`MVC`之类的模式开发。本文我将部分重构插件，以便我们可以对插件进行本地化，并以更简单的方式使用通知。

![](https://raw.githubusercontent.com/qingmei2/qingmei2-blogs-art/master/ide-plugin/part5_1.png)

## 我们要做什么？

今天的目标非常简单，我们将尝试整理插件的代码。为此，我将重点关注两个领域：**通知** 和 **字符串**。

我们将探索一种移除所有字符串硬编码的使用方式，并创建将其移到一个位置的方法，就像我们都知道的`strings.xml`一样，这也将使我们免费进行国际化和本地化。

此外，不同于往常的样板代码，我们还将探索如何在创建新通知时，将相关代码移至`utils`中，我们还将添加在通知中具有超链接的可选项，我们将在下一篇文章中使用它。

与往常一样，本文的所有代码都可以在以下仓库的`Part5`分支中找到。

> https://github.com/marcosholgado/plugin-medium

## 将插件本地化

无论您是否已经阅读了之前的文章，您的插件中都可能有很多字符串被硬编码，像这样：

```kotlin
val lblPassword = JLabel("Token")
```

或者这样

```kotlin
createNotification("Updated","Welcome to the new version", ...)
```

为了摆脱这些硬编码的字符串，我们将利用 **Resource Bundles**，`resource bundle`是一组具有相同基本名称和特定语言的后缀的属性文件。

在`Android`中，您可以通过在`values`文件夹中添加特定语言的后缀来本地化您的应用程序，以便英语可以使用`values-en/strings.xml`，西班牙语可以使用`values-es/strings.xml`，您可将 **Resource Bundles**视为与此等效。

您的`resource bundle`应存在于`resources`文件夹内，我将把它们放在更深层次的`messages`中，完整路径是`resources/messages/`。在其中，我将首先创建一个名为`strings_en.properties`的新属性文件，在该文件中，我将使用以下格式以英文存储字符串。

```
plugin.name = My Jira Plugin
settings.username = Username
settings.token = Token
settings.jiraUrl = Jira URL
settings.regEx = RegEx
...
```

现在我们可以为另一种语言创建另一个属性文件，我对西班牙语非常流利，所以我将创建一个新的`strings_es.properties`文件，其内容如下：

```
plugin.name = Mi Jira Plugin
settings.username = Usuario
settings.token = Token
settings.jiraUrl = Jira URL
settings.regEx = Expresion Regular
...
```
如果我们查看`project`窗口，则可以看到我们的单个`strings_en.properties`文件如何与新的`strings_es.properties`一起捆绑在称为`strings`的 **resource bundle** 中。

![](https://raw.githubusercontent.com/qingmei2/qingmei2-blogs-art/master/ide-plugin/part5_2.png)

要使用新的字符串，我将创建一个`helper`类，该类将获取我们创建的字符串资源，并将基于属性键返回一个字符串。

```kotlin
object StringsBundle {
    @NonNls
    private val BUNDLE_NAME = "messages.strings"
    private var ourBundle: Reference<ResourceBundle>? = null

    private fun getBundle(): ResourceBundle {
        var bundle = SoftReference.dereference(ourBundle)
        if (bundle == null) {
            bundle = ResourceBundle.getBundle(BUNDLE_NAME)
            ourBundle = SoftReference(bundle)
        }
        return bundle!!
    }

    fun message(
        @PropertyKey(
            resourceBundle = "messages.strings"
        ) key: String,
        vararg params: Any
    ): String {
        return CommonBundle.message(getBundle(), key, *params)
    }
}
```

如果现在我们想要检索插件的名称，而不是在需要的地方对字符串进行硬编码，我们可以简单地执行以下操作：

```kotlin
StringsBundle.message("plugin.name")
```

现在是时候使用帮助或我们的`StringsBundle`类，使用适当的本地化字符串替换插件中的所有硬编码字符串。:)

## 通知

到目前为止，我们已经在不同的地方创建了一些通知弹窗，我们一遍又一遍地重复相同的代码，所以我希望简化代码，通过拥有一个`utils`类来处理通知的创建。

在该`utils`类中，我将创建一个新方法来显示通知。此方法将具有不同的参数，例如标题，消息和我们要在哪个项目中显示通知。`Project`可以为空，因为在某些情况下，我们的通知不会显示在`Project`中，例如`Project`未载入的时候。

另一个参数是通知的类型，默认情况下，我们将显示`balloon`类型的通知，这是一种显示类型，而不是通知类型，我们可以显示`INFORMATION`、`WARNING`或`ERROR`类型的通知。

最后一个参数是通知的监听类，您可能已经注意到`balloon`类型通知可以具有一个超链接，如下例所示。

![](https://raw.githubusercontent.com/qingmei2/qingmei2-blogs-art/master/ide-plugin/part5_3.png)

这些超链接就像我们可以根据需要控制的`Action`，而不是将用户重定向到打开浏览器中对应的`URL`，我们可以使用通知监听类指定对应的行为。

最终方法如下所示：

```kotlin
fun createNotification(
        title: String,
        message: String,
        project: Project?,
        type: NotificationType,
        listener: NotificationListener?
) {
    val stickyNotification =
            NotificationGroup(
            "myplugin$title",
                    NotificationDisplayType.BALLOON,
                    true
            )
    stickyNotification.createNotification(
        title, message, type, listener
    ).notify(project)
}
```

现在，您可以使用此方法替换旧代码，并使用在在插件的代码中，最终结果应如下所示：

```kotlin
Utils.createNotification(
        StringsBundle.message("common.success"),
        StringsBundle.message("issue.moved"),
        project,
        NotificationType.INFORMATION,
        null
)
```

## 超链接

由于我已经讨论过使用超链接，因此我认为我可以再深入一点。

我将在我们的`utils`类中添加更多的辅助方法，我们将在下一篇文章中使用。首先，我们将了解如何在通知中集成超链接。

要创建超链接，我们需要一些`html`代码。 我们的通知消息不再是简单的字符串，而是`html`字符串。在`html`代码中，我们将必须使用`<a/>`标签创建一个新的超链接，由于我希望能够复用该代码，因此将整个`html`字符串放入我们先前创建的资源包中。

```
utils.hyperlink.code = <html>{0} <a href=\"postResult.get()\" target=\"blank\">{1}</a> {2}</html>
```

您可以看到我定义了3个不同的部分，我们可以按需修改，第一部分为普通文本，第二部分为超链接文本，最后是第三部分，为普通文本的场景作准备。

通过使用`resource bundle`，我可以快速创建一个新的`utils`方法，该方法将使用给定的超链接字符串前缀，超链接字符串和超链接字符串后缀返回`html`代码。

```kotlin
fun createHyperLink(pre:String, link: String, post: String) =
        StringsBundle.message(
            "utils.hyperlink.code", pre, link, post
        )
```

最后，我将创建一个新的监听`Listener`，使用插件时，您可能要做的主要事情之一是允许用户重新启动`Android Studio`，以完成新版本的安装，或者因为您的插件刚刚进行了更改而需要重启。

新方法返回一个`Listener`，该`Listener`检测何时触发了超链接事件，并在这种情况下重新启动`IDE`。

```kotlin
fun restartListener() =
    NotificationListener { _, event ->
       if (event.eventType === HyperlinkEvent.EventType.ACTIVATED) {
           ApplicationManager.getApplication().restart()
       }
    }
```

以上就是本文的全部内容！现在，您应该能够将插件配置本地化，同时对一些数据进行持久化。

请记住，本文的代码在该系列[GitHub Repo](https://github.com/marcosholgado/plugin-medium/tree/Part4)的`Part5`分支中可用。

在下一篇文章中，我们将介绍模板以及如何使它们在插件中可用，保证用户可以通过简单的右键单击来创建新项目、模块或任何您想要的东西。同时，如果您有任何问题，请随时发表评论或在[Twitter](https://www.twitter.com/orbycius)上关注我。

---
## 《编写AndroidStudio插件》译文系列

* [译: 编写AndroidStudio插件(一):创建一个基本插件](https://github.com/qingmei2/blogs/issues/50)
* [译: 编写AndroidStudio插件(二):持久化数据](https://github.com/qingmei2/blogs/issues/51)
* [译: 编写AndroidStudio插件(三):设置页](https://github.com/qingmei2/blogs/issues/52)
* [译: 编写AndroidStudio插件(四):集成Jira](https://github.com/qingmei2/blogs/issues/53)
* [译: 编写AndroidStudio插件(五):本地化和通知](https://github.com/qingmei2/blogs/issues/54)

## 关于译者

Hello，我是 [却把清梅嗅](https://github.com/qingmei2) ，如果您觉得文章对您有价值，欢迎 ❤️，也欢迎关注我的 [博客](https://blog.csdn.net/mq2553299) 或者 [GitHub](https://github.com/qingmei2)。

如果您觉得文章还差了那么点东西，也请通过 **关注** 督促我写出更好的文章——万一哪天我进步了呢？

* [我的Android学习体系](https://github.com/qingmei2/blogs)
* [关于文章纠错](https://github.com/qingmei2/blogs/blob/main/error_collection.md)
* [关于知识付费](https://github.com/qingmei2/blogs/blob/main/appreciation.md)
* [关于《反思》系列](https://github.com/qingmei2/blogs/blob/main/src/%E5%8F%8D%E6%80%9D%E7%B3%BB%E5%88%97/thinking_in_android_index.md)
