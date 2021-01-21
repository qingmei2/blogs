# [译] 编写AndroidStudio插件(一):创建一个基本插件

> 原文：[Write an Android Studio Plugin Part 1: Creating a basic plugin](https://proandroiddev.com/write-an-android-studio-plugin-part-1-creating-a-basic-plugin-af956c4f8b50)   
作者：[Marcos Holgado](https://medium.com/@marcosholgado)   
译者：[却把清梅嗅](https://github.com/qingmei2)   

早在10月的时候，我就在`Droidcon UK 2018`上针对如何在`Android Studio`上创建自己的插件，以及如何使所有相关操作自动化进行了讨论。因为当时我并没有很多时间对其进行详细介绍，所以这个系列诞生了。

![](https://raw.githubusercontent.com/qingmei2/qingmei2-blogs-art/master/ide-plugin/image.erde8h8rbvk.png)

## 我们要干什么？

本文我们将编写一个非常基本的插件，这次内容也许并不多，但重要的是，我们将学习插件以及创建插件所需的知识。我们还将创建一个新的`Action`，该`Action`将显示一个带有消息的弹出框。

这只是一个非常长的系列文章的第一部分，我将深入研究并为我们的插件扩展更多功能。 您将看到的所有代码都在下方的`GitHub`仓库中：

> [marcosholgado/plugin-medium](https://github.com/marcosholgado/plugin-medium)

随着进度的提升，虽然`master`会一直保持最新的代码，但我还将对每篇文章保留单独的分支。想要如此做，您可以随时返回查看`Part1`分支。

## 第一步：安装 IntelliJ IDEA CE

要创建我们的插件，我们将使用`IntelliJ IDEA`社区版。其主要原因是因为社区版是免费的，且非常容易使用，我们还可以利用`gradle-intellij-plugin`让这个过程变得更加轻松。

我将使用并非`IntelliJ IDEA CE`最新稳定版本的`IntelliJ IDEA CE 2018.1.6`。原因是因为`Android Studio 3.2.1`基于此`IntelliJ`版本，我不也想使用任何可能与`Android Studio`不兼容的新功能。

您可以从此处下载`IntelliJ IDEA CE 2018.1.6`:

> https://www.jetbrains.com/idea/download/previous.html

> 译者注：上述链接会跳转最新的 IDEA Preview 版本，读者可根据自己喜好进行下载。

## 第二步：创建一个新的插件项目

和往常一样，创建一个新的项目。

![image](https://raw.githubusercontent.com/qingmei2/qingmei2-blogs-art/master/ide-plugin/image.jmis4l8g28.png)

选择`Gradle`和`IntelliJ Platform Plugin`，我们唯一要做的，就是确定我们将使用哪种语言。本文我将选择`Kotlin`（`Java`）。

![image](https://raw.githubusercontent.com/qingmei2/qingmei2-blogs-art/master/ide-plugin/image.ng59mbr40ii.png)

这之后，我们需要定义三个属性。

* GroupId — 新项目的groupId，如果您计划在本地部署项目，则可忽略此字段；
* ArtifactId - 新项目的名称；
* Version - 新项目的版本，默认情况下，此字段是自动指定的。

因为我不打算发布此插件，所以只将`myplugin`用作`GroupId`和`ArtifactId`，读者可按需自行定制。

![image](https://raw.githubusercontent.com/qingmei2/qingmei2-blogs-art/master/ide-plugin/image.t9j861ras2.png)

在下一个弹窗，我将不做任何更改，仅保留默认选项。如果您想了解更多有关此步骤的信息，请访问[这里](https://www.jetbrains.com/help/idea/2018.1/gradle.html)。

![image](https://raw.githubusercontent.com/qingmei2/qingmei2-blogs-art/master/ide-plugin/image.xk602rt0kfr.png)

最后一步就是给我们的插件起个名字。

![image](https://raw.githubusercontent.com/qingmei2/qingmei2-blogs-art/master/ide-plugin/image.rv8ggdebull.png)

恭喜你！您刚刚创建了一个新插件，不仅适用于`Android Studio`，而且还适用于基于`IntelliJ`的任何其他`IDE`——尽管您的插件还没有做其它任何事情。

## 第三步：Gradle 和 plugin.xml

现在，我们已经创建了项目，我们将快速浏览两个文件：`plugin.xml`和`build.gradle`。

我们从`plugin.xml`开始。该文件包含一些插件相关的元数据，以及我们必须注册插件不同元素的位置。在必要的时候，我们将更深入地研究此文件中的某些内容。

另一个文件是`build.gradle`，我们已经熟悉它了，所以我不解释它的作用。这是您将获得的默认build.gradle文件（版本不同，内容会略有不同）:

```groovy
plugins {
    id 'org.jetbrains.intellij' version '0.3.12'
    id 'org.jetbrains.kotlin.jvm' version '1.3.10'
}

group 'myplugin'
version '1.0-SNAPSHOT'

repositories {
    mavenCentral()
}

dependencies {
    compile "org.jetbrains.kotlin:kotlin-stdlib-jdk8"
}

compileKotlin {
    kotlinOptions.jvmTarget = "1.8"
}
compileTestKotlin {
    kotlinOptions.jvmTarget = "1.8"
}
intellij {
    version '2018.2.2'
}
patchPluginXml {
    changeNotes """
      Add change notes here.<br>
      <em>most HTML tags may be used</em>"""
}
```

一切似曾相识，因为我们习惯了`Android`项目。当我们开始添加更多依赖项时，我们将折回该文件，依此类推，但目前我只关注两件事。

首先，您可以看到我们的 **依赖项** 使用了`compile`而不是`implementation/api`，主要原因是`gradle-intellij-plugin`尚不支持`implementation/api`。

> 译者注：最新版本的 IDEA 插件已支持。

不过，如果您需要`kapt`进行任何类型的注解处理，则可以使用`kapt`（说的就是你，`Dagger`）。与任何`Android`项目中一样，您可以使用所需的任何库，但使用时务必小心，因为这不是`Android`项目。

其次，我们有一个新的部分称为`intellij`。 在这里，我们将在需要时添加更多属性，如插件依赖性等。现在，我们唯一拥有的属性是`version`，它指定了应该用作依赖性的`IDEA`的发行版本。

在继续之前，我将向`intellij`部分添加另一个属性。 我们现在拥有的插件只能在`IntelliJ IDEA CE`上调试。 如果我们想立即调试插件的行为，一个新的`IntelliJ`实例将被启动，我们将不得不在那里进行调试/测试。显然我们想在`Android Studio`上测试我们的插件，以便告诉`gradle-intellij-plugin`我们要使用`Android Studio`，我们必须添加一个新属性。

最终该部分声明如下：

```groovy
intellij {
    version '2018.1.6'
    alternativeIdePath '/Applications/Android Studio.app'
}
```

通过使用`AlternativeIdePath`并指向本地安装的`Android Studio`，我们告诉`gradle-intellij-plugin`每当运行插件或对其进行调试时都使用`Android Studio`，而不是使用默认的`IntelliJ IDE`。

如果您迫不及待要其他文章来查看可使用的其他属性，请访问[这里](https://github.com/JetBrains/gradle-intellij-plugin)以获取更多信息。

## 第四步：编写第一个Action

我们可以在插件中使用不同的元素，我们将在以后的文章中看到所有这些元素，但现在我们将重点介绍最常用的：**actions**。

当您单击工具栏或菜单项时，基本上就是一个动作，就这么简单，下列图片中您看到的所有内容都是`Actions`展示的：

![image](https://raw.githubusercontent.com/qingmei2/qingmei2-blogs-art/master/ide-plugin/image.q73gxz0avl.png)

通过`Actions`，我们可以将自己的项目应用到`Android Studio`的菜单和工具栏上，这些`Action`被组织成`Group`，这些`Group`可以包含其他`Group`，依此类推。

要创建一个新的`Action`，我们必须让这个类继承`AnAction`并重写`actionPerformed`方法。

```java
class MyAction: AnAction() {

    override fun actionPerformed(e: AnActionEvent) {
        val noti = NotificationGroup("myplugin", NotificationDisplayType.BALLOON, true)
        noti.createNotification("My Title",
                                "My Message",
                                NotificationType.INFORMATION,
                                null
                               ).notify(e.project)
    }
}
```

在这里，我们创建了一个简单的`Action`，它将显示一个带有标题和消息的弹窗。我们要做的最后一件事是将此`Action`添加到我们的`plugin.xml`文件中。

```xml
<actions>
    <group id="MyPlugin.TopMenu"
           text="_MyPlugin"
           description="MyPlugin Toolbar Menu">
        <add-to-group group-id="MainMenu" anchor="last"/>
        <action id="MyAction"
                class="actions.MyAction"
                text="_MyAction"
                description="MyAction"/>
    </group>
</actions>
```

我们的`Action`必须属于一个`Group`，因此，首先，我创建了一个`ID`为`MyPlugin.TopMenu`的新`Group`。 我还将该`Group`添加到`MainMenu Group`，它是您可以在任何`IntelliJ IDE`上看到的主要工具栏。我将其位置锚定为最后一个，这样我们的`Group`将处于该`Group`的最后位置。 最后，我将`Action`添加到`MyPlugin.TopMenu Group`中，以便我们可以从那里访问它。

如果您想知道我如何知道`MainMenu ID`的存在，只需命令+单击`MainMenu ID`，它将带您进入一个名为`PlatformActions.xml`的文件，该文件包含绝大多数`Action`（类似于`VcsActions.xml`） 和`IDE`中的`Group`。

我们可以对`Action`和`Group`执行许多不同的操作，例如添加分隔符或复用它们。我将在以后的文章中对其进行探讨，但现在您可以在[这里](http://www.jetbrains.org/intellij/sdk/docs/basics/action_system.html)查看它们。

## 第五步：运行它！

![image](https://raw.githubusercontent.com/qingmei2/qingmei2-blogs-art/master/ide-plugin/image.gty750js4wh.png)

就这样，我们刚刚编写了一个非常简单的插件。现在我们可以使用`run`按钮运行它，也可以对其进行调试。这将创建一个新的`Android Studio`实例，它将安装我们的插件。另一个选项是运行`buildPlugin gradle`任务，该任务将生成一个`.jar`文件，您可以将其作为插件安装在`Android Studio`或任何其他`IntelliJ IDE`上。

安装插件并运行`Android Studio`后，您现在可以在主工具栏上看到包含`MyAction`的新`MyPlugin group`。

![image](https://raw.githubusercontent.com/qingmei2/qingmei2-blogs-art/master/ide-plugin/image.uauu1hpax9i.png)

在单击`MyAction`之后，将显示一个新的弹窗，其包含您定义的标题和消息。请记住在`log`日志窗口上启用 **Show balloons**，否则，您将不会看到弹窗，而是一个日志事件。

![image](https://raw.githubusercontent.com/qingmei2/qingmei2-blogs-art/master/ide-plugin/image.jky23byf9c.png)

第一部分就是这样。在[第二部分](https://proandroiddev.com/write-an-android-studio-plugin-part-2-persisting-data-5f81ca5d9e43)，我们将研究如何使用组件存储数据并保存插件的状态。同时，如果您有任何疑问，请访问[twitter](https://www.twitter.com/orbycius)或发表评论。

如果您想观看我在`Droidcon UK`上发表的演讲，请点击[这里](https://skillsmatter.com/skillscasts/12166-write-your-own-android-studio-plugin-and-automate-everything)。

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
