# [译] 编写AndroidStudio插件(三):设置页

> 原文：[Write an Android Studio Plugin Part 3: Settings](https://proandroiddev.com/write-an-android-studio-plugin-part-3-settings-662a535c6962)   
> 作者：[Marcos Holgado](https://medium.com/@marcosholgado)   
> 译者：[却把清梅嗅](https://github.com/qingmei2)   
>《编写AndroidStudio插件》系列是 IntelliJ IDEA 官方推荐的学习IDE插件开发的博客专栏，希望对有需要的读者有所帮助。

在本系列的[第二部分](https://proandroiddev.com/write-an-android-studio-plugin-part-2-persisting-data-5f81ca5d9e43)中，我们学习了如何使用`Component`对数据进行持久化，以及通过这些数据，在用户更新我们的插件后展示更新了哪些新功能。在今天的文章中，我们将看到如何使用持久化的数据来创建设置页面。

![](https://raw.githubusercontent.com/qingmei2/qingmei2-blogs-art/master/ide-plugin/part3_1.jpeg)

请记住，您可以在`GitHub`上找到本系列的所有代码，还可以在对应的分支上查看每篇文章的相关代码，本文的代码在`Part3`分支中。

> https://github.com/marcosholgado/plugin-medium

## 我们要做什么？

本文的目的是为我们的插件创建一个 **设置页面**，这将是我们迈向将`JIRA`搬运过来的第一步。我们的设置页面上只会有一个用户名和密码字段，我们的插件将使用该用户名和密码字段与`Jira API`进行交互。我们还希望能够为 **每个项目** 分别配不同的设置，从而允许用户根据项目使用不同的`Jira`帐户（这可能很有用）。

## 第一步：新建一个Project级别的Component

在本系列的第二部分中，我们已经了解了什么是`Component`，并且还了解了存在三种不同类型的`Component`。 因为我们希望能够根据我们的`Android Studio`的各`Project`进行不同的设置，因此显而易见的选择是创建一个新的`Project Component`。

我们基本上是在复制和粘贴我们先前创建的`Component`，但是删除了所有不必要的方法并添加了两个新字段。这些字段将是`public`的，因为我们将在插件的其它部分中使用它们。

另一处不同是这次我们实现`ProjectComponent`接口并实现`AbstractProjectComponent`方法，当然，它的构造方法中也有一个`project`参数。最后，我们有一个`companion object`，通过一个`project`参数，以获取我们的`JiraComponent`的实例。这将使我们能够从插件中其他位置访问存储的数据。新的`JiraComponent`看起来像这样：

```kotlin
@State(name = "JiraConfiguration",
        storages = [Storage(value = "jiraConfiguration.xml")])
class JiraComponent(project: Project? = null) :
        AbstractProjectComponent(project),
        Serializable,
        PersistentStateComponent<JiraComponent> {

    var username: String = ""
    var password: String = ""

    override fun getState(): JiraComponent? = this

    override fun loadState(state: JiraComponent) =
            XmlSerializerUtil.copyBean(state, this)

    companion object {
        fun getInstance(project: Project): JiraComponent =
                project.getComponent(JiraComponent::class.java)
    }
}
```

如我们在上文所做的一样，我们还必须在`plugin.xml`文件中注册`Component`：

```xml
<project-components>
    <!-- Add your project components here -->
    <component>
        <implementation-class>
            components.JiraComponent
        </implementation-class>
    </component>
</project-components>
```

## 第二步：UI

在针对我们的设置页面进行下一步之前，我们需要了解如何通过使用`Java Swing`在`IntelliJ`上创建`UI`。`IntelliJ`有许多可以使用的`Swing`组件，以保证插件`UI`与`IDE`中其它插件保持一致。 但不要被名字中带有`Java`给欺骗了，因为您仍可将代码转换为`Kotlin`。

创建新`GUI`（图形用户界面）的一种方法是，只需右键单击并转到`New`，然后单击`GUI Form`。该操作将创建一个名为`YourName.form`的新文件，该文件将链接到另一个名为`YourName.java`的文件。相比于按照`IntelliJ`给的编辑器模式进行开发，我更喜欢用我自己的方式，给一个提示：

![](https://raw.githubusercontent.com/qingmei2/qingmei2-blogs-art/master/ide-plugin/part3_2.jpeg)

**我将会使用 Eclipse !（欢呼声）**

我知道你在想什么，但老实说，它真的很棒。由于一些原因，`IntelliJ`的编辑器确实很难用，我无法获得预期的效果，但是，如果你对`IntelliJ`感到满意，请继续使用它！

> **译者注**：我也并不喜欢`IDEA`官方的编辑器，但也没有很大必要去使用`Eclipse`，因为使用`Eclipse`只是对UI预览而已。

回到`Eclipse`，您可以从[这里](https://www.eclipse.org/downloads/)下载它。 我目前有`Oxygen.3a`版本，该版本有些旧，但是对于我们要做的并不重要。只需创建一个新项目，然后右键单击`New`，`Other`，然后选择`JPanel`：

![](https://raw.githubusercontent.com/qingmei2/qingmei2-blogs-art/master/ide-plugin/part3_3.png)

下图是我们的设置页预览：

![](https://raw.githubusercontent.com/qingmei2/qingmei2-blogs-art/master/ide-plugin/part3_4.png)

接下来我们只需将创建好的源代码从`Eclipse`复制过来就行了，然后就可以关闭`Eclipse`了。

回到我们的插件，我们现在将创建一个名为`settings`的新包和一个名为`JiraSettings`的新类。 在该类中，我们将创建一个名为`createComponent()`的新方法，最后我们可以在该方法中粘贴从`Eclipse`复制的源​​代码。然后是时候将代码转换为`Kotlin`，您应该也可以自动将其成功转换为`Kotlin`。

完成所有这些操作后，您可能会遇到一些错误，因此请修复它们。

我们需要解决的第一件事是我们的`createComponent()`方法必须返回一个`JComponent`，具体原因接下来我们会说到。

因为`Eclipse`假定我们已经在`JPanel`中，所以您可以看到很多`add`方法或似乎并不存在的方法，原因是因为我们不在`JPanel`中。为解决该问题，我们必须创建一个新的`JPanel`并给它一些边界（您可以从在`Eclipse`中创建的`JPanel`中获取值），并且由于`JPanel`是`JComponent`的子类，因此我们将在我们的方法中将其返回。

最后，我们只需要进行一些调整就可以编译整个程序，最终效果应该如下：

```kotlin
class JiraSettings {

    private val passwordField = JPasswordField()
    private val txtUsername = JTextField()

    fun createComponent(): JComponent {

        val mainPanel = JPanel()
        mainPanel.setBounds(0, 0, 452, 120)
        mainPanel.layout = null

        val lblUsername = JLabel("Username")
        lblUsername.setBounds(30, 25, 83, 16)
        mainPanel.add(lblUsername)

        val lblPassword = JLabel("Password")
        lblPassword.setBounds(30, 74, 83, 16)
        mainPanel.add(lblPassword)

        passwordField.setBounds(125, 69, 291, 26)
        mainPanel.add(passwordField)

        txtUsername.setBounds(125, 20, 291, 26)
        mainPanel.add(txtUsername)
        txtUsername.columns = 10

        return mainPanel
    }
}
```

## 第三步：Extensions 和 Extension points

在继续开发设置页面前，我们必须讨论`extensions`和`extension points`。它们将允许您的插件与其他插件或与`IDE`本身进行交互。

* 如果要扩展其他插件或`IDE`的功能，则必须声明一个或多个`extensions`。
* 如果要让插件允许其他插件扩展其功能，则必须声明一个或多个`extension points`。

因为我们要将设置页面添加到`Android Studio`的`Preferences`中，所以我们真正要做的是扩展`Android Studio`的功能，因此我们必须声明一个`extensions`。

为此，我们必须实现`Configurable`，同时还必须重写一些方法。

* 幸运的是，我们已经有了`createComponent()`方法，因此我们只需添加`override`关键字就可以了。
* 我们将创建一个`boolean`的`modified`，其默认值为`false`，并作为`isModified()`的返回值。我们稍后会再讲到这一点，目前它代表了设置页面`apply`按钮是否被启用。
* 我们将`getDisplayName()`的返回值用于展示设置页的名字。
* 在`apply`方法中，我们需要编写将在用户单击`Apply`时执行的代码。很简单，我们为用户所在的`Project`获取`JiraComponent`的实例，然后将`UI`中的值保存到`Component`中。最后，我们将`Modify`设置为`false`，届时我们要禁用`Apply`按钮。

最终展示效果如下：

```kotlin
class JiraSettings(private val project: Project): Configurable {

    private val passwordField = JPasswordField()
    private val txtUsername = JTextField()

    private var modified = false

    override fun isModified(): Boolean = modified

    override fun getDisplayName(): String = "MyPlugin Jira"

    override fun apply() {
        val config = JiraComponent.getInstance(project)
        config.username = txtUsername.text
        config.password = String(passwordField.password)

        modified = false
    }

    override fun createComponent(): JComponent {

        val mainPanel = JPanel()
        mainPanel.setBounds(0, 0, 452, 120)
        mainPanel.layout = null

        val lblUsername = JLabel("Username")
        lblUsername.setBounds(30, 25, 83, 16)
        mainPanel.add(lblUsername)

        val lblPassword = JLabel("Password")
        lblPassword.setBounds(30, 74, 83, 16)
        mainPanel.add(lblPassword)

        passwordField.setBounds(125, 69, 291, 26)
        mainPanel.add(passwordField)

        txtUsername.setBounds(125, 20, 291, 26)
        mainPanel.add(txtUsername)
        txtUsername.columns = 10

        return mainPanel
    }
}
```

## 第四步：解决最后的问题

我们几乎完成了，只剩下最后几个问题。

首先，我们要保存用户的偏好设置，但目前我们还未对其加载。`UI`是在`createComponent()`方法中创建的，因此我们只需要在返回之前添加以下代码，即可使用先前存储的值设置`UI`:

```kotlin
val config = JiraComponent.getInstance(project)
txtUsername.text = config.username
passwordField.text = config.password
```

接下来，我们将使用`isModified()`解决问题。 当用户修改设置页中的任何值时，我们需要以某种方式将值从`false`更改为`true`。一种非常简单的方法是实现 **DocumentListener**，该接口为我们提供了3种方法： **changeUpdate** ，**insertUpdate** 和 **removeUpdate**。

在这些方法中，我们唯一要做的就是简单地将`Modify`的值更改为`true`，最后将`DocumentListener`添加到我们的密码和用户名字段中。

```kotlin
override fun changedUpdate(e: DocumentEvent?) {
    modified = true
}

override fun insertUpdate(e: DocumentEvent?) {
    modified = true
}

override fun removeUpdate(e: DocumentEvent?) {
    modified = true
}
```

最终实现如下：

```kotlin
class JiraSettings(private val project: Project): Configurable, DocumentListener {
    private val passwordField = JPasswordField()
    private val txtUsername = JTextField()
    private var modified = false

    override fun isModified(): Boolean = modified

    override fun getDisplayName(): String = "MyPlugin Jira"

    override fun apply() {
        val config = JiraComponent.getInstance(project)
        config.username = txtUsername.text
        config.password = String(passwordField.password)
        modified = false
    }

    override fun changedUpdate(e: DocumentEvent?) {
        modified = true
    }

    override fun insertUpdate(e: DocumentEvent?) {
        modified = true
    }

    override fun removeUpdate(e: DocumentEvent?) {
        modified = true
    }

    override fun createComponent(): JComponent {

        val mainPanel = JPanel()
        mainPanel.setBounds(0, 0, 452, 120)
        mainPanel.layout = null

        val lblUsername = JLabel("Username")
        lblUsername.setBounds(30, 25, 83, 16)
        mainPanel.add(lblUsername)

        val lblPassword = JLabel("Password")
        lblPassword.setBounds(30, 74, 83, 16)
        mainPanel.add(lblPassword)

        passwordField.setBounds(125, 69, 291, 26)
        mainPanel.add(passwordField)

        txtUsername.setBounds(125, 20, 291, 26)
        mainPanel.add(txtUsername)
        txtUsername.columns = 10

        val config = JiraComponent.getInstance(project)
        txtUsername.text = config.username
        passwordField.text = config.password

        passwordField.document?.addDocumentListener(this)
        txtUsername.document?.addDocumentListener(this)

        return mainPanel
    }
}
```

## 第五步：声明 extension

与`Component`相同，我们还必须在`plugin.xml`文件中声明`extension`。

```xml
<extensions defaultExtensionNs="com.intellij">
    <defaultProjectTypeProvider type="Android"/>
    <projectConfigurable
            instance="settings.JiraSettings">
    </projectConfigurable>
</extensions>
```

大功告成！调试或安装插件时，您可以转到`Android Studio`中的`Preferences/Other Settings`，找到新的设置页。您也可以使用不同的`Project`进行测试，并且每个`Project`都会记住自身的设置。

![](https://raw.githubusercontent.com/qingmei2/qingmei2-blogs-art/master/ide-plugin/part3_5.png)

这就是第三部分的全部内容。在下一篇文章中，我们将看到如何使用这些设置来创建新的`Action`，将`Jira`相关功能迁移过来。同时，如果您有任何疑问，请访问[Twitter](https://www.twitter.com/orbycius)或发表评论。

---
## 《编写AndroidStudio插件》译文系列

* [译: 编写AndroidStudio插件(一):创建一个基本插件](https://github.com/qingmei2/blogs/issues/50)
* [译: 编写AndroidStudio插件(二):持久化数据](https://github.com/qingmei2/blogs/issues/51)
* [译: 编写AndroidStudio插件(三):设置页](https://github.com/qingmei2/blogs/issues/52)
* [译: 编写AndroidStudio插件(四):整合Jira](https://github.com/qingmei2/blogs/issues/53)
* [译: 编写AndroidStudio插件(五):本地化和通知](https://github.com/qingmei2/blogs/issues/54)

## 关于译者

Hello，我是 [却把清梅嗅](https://github.com/qingmei2) ，如果您觉得文章对您有价值，欢迎 ❤️，也欢迎关注我的 [博客](https://blog.csdn.net/mq2553299) 或者 [GitHub](https://github.com/qingmei2)。

如果您觉得文章还差了那么点东西，也请通过 **关注** 督促我写出更好的文章——万一哪天我进步了呢？

* [我的Android学习体系](https://github.com/qingmei2/blogs)
* [关于文章纠错](https://github.com/qingmei2/blogs/blob/main/error_collection.md)
* [关于知识付费](https://github.com/qingmei2/blogs/blob/main/appreciation.md)
* [关于《反思》系列](https://github.com/qingmei2/blogs/blob/main/src/%E5%8F%8D%E6%80%9D%E7%B3%BB%E5%88%97/thinking_in_android_index.md)
