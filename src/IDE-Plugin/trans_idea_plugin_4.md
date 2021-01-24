# [译] 编写AndroidStudio插件(四):集成Jira

> 原文：[Write an Android Studio Plugin Part 4: Jira Integration](https://proandroiddev.com/write-an-android-studio-plugin-part-4-jira-integration-cd54df01cff6)   
> 作者：[Marcos Holgado](https://medium.com/@marcosholgado)   
> 译者：[却把清梅嗅](https://github.com/qingmei2)   
>《编写AndroidStudio插件》系列是 IntelliJ IDEA 官方推荐的学习IDE插件开发的博客专栏，希望对有需要的读者有所帮助。

在本系列的第三部分中，我们学习了如何使用`Component`对数据进行持久化，并利用这些数据来创建新的设置页面。在今天的文章中，我们将使用这些数据将`Jira`与我们的插件快速集成在一起。

![](https://raw.githubusercontent.com/qingmei2/qingmei2-blogs-art/master/ide-plugin/part4_1.png)

请记住，您可以在`GitHub`上找到本系列的所有代码，还可以在对应的分支上查看每篇文章的相关代码，本文的代码在`Part4`分支中。

> https://github.com/marcosholgado/plugin-medium

## 我们要做什么？

今天这篇文章的目的是解释如何将第三方`API`和库集成到插件中。我将应用一个简单的`MVP`模式，您可以更改为`MVC`或任何您喜欢的开发模式。

今天，我们将`Jira`集成到我们的插件中，我们要做的是能够在`Android Studio`中将`Jira Scrum`板上的`issue`移至下一栏。因为`Jira`的`issue ID`将基于我们当前的`git`分支，所以我们的插件将从我们当前的分支中解析`issue ID`，而不是强迫用户手动输入或从其他位置选择`issue ID`。

在开始前，我们先进行一些假设。

* 1、在移动`issue`（比如记录时间等）之前，您无需填写任何必填字段，否则你的`UI`将需要额外的字段供用户输入；
* 2、我们将使用[`Jira Cloud Platform API v3`](https://developer.atlassian.com/cloud/jira/platform/rest/v3/intro/)；
* 3、为了简单起见，我们还将使用 [基本身份验证(`Basic auth`)](https://developer.atlassian.com/cloud/jira/platform/basic-auth-for-rest-apis/) ，因为我们的目标是学习如何集成第三方工具，而不是如何在`Jira`中进行正确的身份验证。

> 除非您正在构建仅供内部使用的工具（例如脚本和机器人），否则我们（`Jira`）不建议使用基本身份验证。

这就是我的面板页在`Jira`中展示出来的样子，`issue`只能向前推进，并且只能从一列移至下一列，您不能跳过流程中的任何一列。

![](https://raw.githubusercontent.com/qingmei2/qingmei2-blogs-art/master/ide-plugin/part4_2.png)

## 第一步

首先，我们将在设置页面中添加更多字段。根据我们的目标，我们需要添加一个新的`regex`字段，其中将包含一个正则表达式以从当前分支中提取`issue ID`。我们还将需要一个`Jira URL`字段，该字段将用作`API`调用的基本`URL`。最后，`Jira API`需要使用`auth`令牌而不是密码，为了清楚起见，我将旧密码字段的名称更改为`token`。

这些变动应该简单直观，如下所示：

```kotlin
@State(name = "JiraConfiguration",
        storages = [Storage(value = "jiraConfiguration.xml")])
class JiraComponent(project: Project? = null) :
        AbstractProjectComponent(project),
        Serializable,
        PersistentStateComponent<JiraComponent> {

    var username: String = ""
    var token: String = ""
    var url: String = ""
    var regex: String = ""

    override fun getState(): JiraComponent? = this

    override fun loadState(state: JiraComponent) =
            XmlSerializerUtil.copyBean(state, this)

    companion object {
        fun getInstance(project: Project): JiraComponent =
                project.getComponent(JiraComponent::class.java)
    }
}
```

```kotlin
class JiraSettings(private val project: Project): Configurable, DocumentListener {
    private val tokenField: JPasswordField = JPasswordField()
    private val txtUsername: JTextField = JTextField()
    private val txtUrl: JTextField = JTextField()
    private val txtRegEx: JTextField = JTextField()

    private var modified = false

    override fun isModified(): Boolean = modified

    override fun getDisplayName(): String = "MyPlugin Jira"

    override fun apply() {
        val config = JiraComponent.getInstance(project)
        config.username = txtUsername.text
        config.token = String(tokenField.password)
        config.url = txtUrl.text
        config.regex = txtRegEx.text

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
        mainPanel.setBounds(0, 0, 452, 254)
        mainPanel.layout = null

        val lblUsername = JLabel("Username")
        lblUsername.setBounds(30, 25, 83, 16)
        mainPanel.add(lblUsername)

        val lblPassword = JLabel("Token")
        lblPassword.setBounds(30, 74, 83, 16)
        mainPanel.add(lblPassword)

        val lblUrl = JLabel("Jira URL")
        lblUrl.setBounds(30, 123, 83, 16)
        mainPanel.add(lblUrl)

        val lblRegEx = JLabel("RegEx")
        lblRegEx.setBounds(30, 172, 83, 16)
        mainPanel.add(lblRegEx)

        txtUsername.setBounds(125, 20, 291, 26)
        txtUsername.columns = 10
        mainPanel.add(txtUsername)

        tokenField.setBounds(125, 69, 291, 26)
        mainPanel.add(tokenField)

        txtUrl.setBounds(125, 118, 291, 26)
        txtUrl.columns = 10
        mainPanel.add(txtUrl)

        txtRegEx.setBounds(125, 167, 291, 26)
        txtRegEx.columns = 10
        mainPanel.add(txtRegEx)

        val config = JiraComponent.getInstance(project)
        txtUsername.text = config.username
        tokenField.text = config.token
        txtUrl.text = config.url
        txtRegEx.text = config.regex

        tokenField.document?.addDocumentListener(this)
        txtUsername.document?.addDocumentListener(this)
        txtUrl.document?.addDocumentListener(this)
        txtRegEx.document?.addDocumentListener(this)

        return mainPanel
    }
}
```

在进行下一步之前，请确保这些更改确实有效，并且新字段的数据已正确进行了保存。

## 第二步：您仍在编写代码！

现在，我们可以创建一个新的`Action`，将所有代码插入其中，然后转到下一篇文章，但我们不会这样做，因为：

> 您仍在编写代码！ -Marcos Holgado（就是我！）

仅仅因为这是一个插件，并不意味着您不必对其进行维护或遵循任何编码规范。我看到太多插件，其中所有代码都在一个`Action`中。我不明白，如果您不想将所有代码都放在`Action`中，为什么还要这么做？

今天，我将使用`MVP`模式，但您可以使用任何您喜欢的模式，只要您遵循经典的编码规范，就不会有太大的不同。我们的看起来像这样：

![](https://raw.githubusercontent.com/qingmei2/qingmei2-blogs-art/master/ide-plugin/part4_3.png)

我们的`JiraMoveAction`将创建一个新的`JiraMoveDialog`，它将具有一个`JiraMoveDialogPresenter`，该`JiraMoveDialogPresenter`将与`Model`，网络等进行通信。此外，`JiraMoveDialog`将创建一个`JiraMovePanel`，其唯一原因是要分离更多的`UI`层，我将解释说在步骤4中。

除此之外，我们将使用`Retrofit`将对`Jira API`和`Dagger2`的网络请求用作`DI`框架（您知道我有点喜欢`Dagger`）。

首先，我们将在`action package`中创建一个新的`package`，以将`Action`与其余代码进一步分开，然后创建所有提及的文件。

![](https://raw.githubusercontent.com/qingmei2/qingmei2-blogs-art/master/ide-plugin/part4_4.png)

## 第三步：Models

我们的`Model`将非常简单，因为我们只需要处理`Transition`，并且由于不处理在`Transition`的必填字段。我们唯一需要的信息是`Transition id`和`Transition name`。如果您需要配置其他东西（例如添加评论），请点击[这里](https://developer.atlassian.com/cloud/jira/platform/rest/v3/#api-api-3-issue-issueIdOrKey-transitions-get)查看文档。

我将把这些`Model`放在`network`包下的`Models.kt`文件中，实现如下：

```kotlin
data class Transition(val id: String, val name: String = "") {
    override fun toString(): String = name
}

data class TransitionsResponse(val transitions: List<Transition>)

data class TransitionData(val transition: Transition)
```

## 第四步：JiraMovePanel

顾名思义，该文件将成为具有所需`UI`的`JPanel`。我们不会在此处创建任何 **OK** 或 **Cancel** 按钮，因为这将作为`JiraMoveDialog`的一部分出现，但我将在下一小节中进行讨论。

现在，我们将创建一个非常简单的`UI`，其包含一个`combobox`（用于显示可用的`Transition`）和一个` text field`（用于显示`issue ID`），这个`text field`让用户根据需要手动进行配置。

我们的`JiraMovePanel`类继承自`JPanel`，根据上文，我们将在`Eclipse`中创建`UI`，复制粘贴代码并将其转换为`Kotlin`。

与我们在设置页面上所做的操作相比，存在一些差异，这是因为我们继承了`JPanel`，我们已经在`Panel`中，因此我们可以直接调用`add()`。

我们还必须重写`getPreferredSize()`来设置`Panel`的大小，不要忘记这样做！

最后，我添加了一些方法，这些方法将从`JiraMoveDialog`中调用以更改字段的值，最终文件如下所示：

```kotlin
class JiraMovePanel : JPanel() {

    private val comboTransitions = ComboBox<Transition>()
    val txtIssue = JTextField()

    init {
        initComponents()
    }

    private fun initComponents() {
        layout = null

        val lblJiraTicket = JLabel("Issue")
        lblJiraTicket.setBounds(25, 33, 77, 16)
        add(lblJiraTicket)

        txtIssue.setBounds(114, 28, 183, 26)
        add(txtIssue)

        val lblTransition = JLabel("Transition")
        lblTransition.setBounds(25, 75, 77, 16)
        add(lblTransition)

        comboTransitions.setBounds(114, 71, 183, 27)
        add(comboTransitions)
    }

    override fun getPreferredSize() = Dimension(300, 110)

    fun addTransition(transition: Transition) = comboTransitions.addItem(transition)

    fun setIssue(issue: String) {
        txtIssue.text = issue
    }

    fun getTransition() : Transition = comboTransitions.selectedItem as Transition
}
```

## 第五步：展示UI

让我们确保刚才所创用户界面显示的正确性，我们首先需要将`JiraMoveDialog`与`JiraMovePanel`链接起来，因此我们来实现`JiraMoveDialog`。

首先，我们需要继承`DialogWrapper`，`IntelliJ`提供了这个包装器，我们应该将其用于插件中的所有模式的对话框。`IntelliJ`还提供了一些免费功能，例如`OK`或`Cancel`按钮，因此我们不必在`JPanel`中创建它们。

我们还必须重写`createCenterPanel()`以返回刚刚创建的`Panel`，并在初始化对话框时调用`init()`。现在，这是我们的`JiraMoveDialog`类：

```kotlin
class JiraMoveDialog constructor(val project: Project):
        DialogWrapper(true) {

    init {
        init()
    }

    override fun createCenterPanel(): JComponent? {
        return JiraMovePanel()
    }
}
```

现在，我们可以在我们的`JiraMoveAction`中创建一个新对话框，这对您现在来说应该很简单，因此我不再赘述。

```kotlin
class JiraMoveAction : AnAction() {

    override fun actionPerformed(event: AnActionEvent) {
        val dialog = JiraMoveDialog(event.project!!)
        dialog.show()
    }
}
```

最后一步是将新的`Action`添加到`plugin.xml`文件中。

您现在可以调试插件，执行`Action`，并且应该看到以下的内容：

![](https://raw.githubusercontent.com/qingmei2/qingmei2-blogs-art/master/ide-plugin/part4_5.png)


## 第六步：DI 和获取 Git 分支

到目前为止，我们已经完成了`UI`工作，让我们开始使用`Presenter`并将`Dagger2`集成到项目中。

> 注意：如果不需要，您不必使用`Dagger`，但我建议像在其他任何项目中一样，使用某种形式的依赖注入。

要添加`Dagger`，我们只需像通常那样，在`build.gradle`文件中添加依赖项即可。别忘了也添加`kapt`。请注意，由于`intellij-gradle-plugin`尚不支持`implementation`或`api`，因此我们尚未使用它们。

```groovy
apply plugin: 'kotlin-kapt'
dependencies {
    compile 'org.jetbrains.kotlin:kotlin-stdlib-jdk8:1.3.10'

    compile 'com.google.dagger:dagger:2.20'
    kapt 'com.google.dagger:dagger-compiler:2.20'
}
```

我们还希望从`IDE`中获取当前分支，为此，我们需要添加插件依赖。每当您需要第三方插件依赖（例如`android`插件或任何其他插件）时，都必须将该插件添加到`gradle`文件的插件列表中。在我们的例子中，我们将需要`git4idea`插件。

```groovy
intellij {
    version '2018.1.6'
    plugins = ['git4idea']
    alternativeIdePath '/Applications/Android Studio.app'
}
```

我们还必须在`plugin.xml`文件中添加依赖。

```xml
<depends>Git4Idea</depends>
```

添加了所有依赖后，我们可以专注于依赖注入。首先，我将创建一个新的`Dagger Component`以及一个`Module`，将来它将帮助我们进行测试，并使我们的架构更整洁。

现在，我们将只注入我们正在处理的`project`，即`JiraMoveDialog`（`View`层）和保存我们的设置的`JiraComponent`。 我还将`Component`命名为`JiraDIComponent`，因此我们不会把它和用于保存设置的`JiraComponent`相混淆。

```kotlin
@Component(modules = [JiraModule::class])
interface JiraDIComponent {
    fun inject(jiraMoveDialog: JiraMoveDialog)
}
```

```kotlin
@Module
class JiraModule(
        private val view: JiraMoveDialog,
        private val project: Project
) {
    @Provides
    fun provideView() : JiraMoveDialog = view

    @Provides
    fun provideProject() : Project = project

    @Provides
    fun provideComponent() : JiraComponent =
            JiraComponent.getInstance(project)
}
```

> 如果您对依赖注入或Dagger不熟悉，建议您看一下Jake Wharton的演讲：
> https://www.youtube.com/watch?v=plK0zyRLIP8

现在，我们可以使用`Dagger`创建`Presenter`并将所需一切注入到构造函数中,要获得当前正在使用的分支实际上非常简单，只需从当前项目中获得一个`repository manager`即可。 通常，您将有一个`repository`，因此您只需调用`first()`并获取当前分支的名称。之后，通过使用存储在我们设置中的正则表达式，我们可以匹配并找到`Jira`中`issue`的`ID`。

我将在设置中存储的正则表达式为`[a-zA-Z] +-[0-9] +`，因为`Jira ID`的格式为`Project-Number`（即`DROID-12`），并且我为分支命名作为`DROID-12-this-is-a-bug`。

```kotlin
class JiraMoveDialogPresenter @Inject constructor(
        private val view: JiraMoveDialog,
        private val project: Project,
        private val component: JiraComponent
) {

    fun load() {
        getBranch()
    }

    private fun getBranch() {
        val repositoryManager = GitRepositoryManager.getInstance(project)
        val repository = repositoryManager.repositories.first()
        val ticket = repository.currentBranch!!.name
        val match = Regex(component.regex).find(ticket)
        match?.let {
            view.setIssue(match.value)
        }
    }
}
```

回到`JiraMoveDialog`，我们必须注入`Presenter`，并实现`setIssue()`方法以根据`Git`分支更改字段的值。为此，我们将创建一个`JPanel`的变量，而非在`createCenterPanel()`上返回新的`JPanel`，然后可以使用该`Panel`来更改字段的值。

我将`isModal`设置为`true`。每当我们将`modal`设置为`true`时，我们都会阻止`UI`，因此用户必须退出我们的对话框才能再次与`IDE`交互，如果需要，可以随意更改该值。然后，我们调用`presenter.load()`从`IDE`中获取分支，和之前一样，我们还须调用`init()`。

```kotlin
class JiraMoveDialog constructor(project: Project):
        DialogWrapper(true) {

    @Inject
    lateinit var presenter: JiraMoveDialogPresenter
    private val panel : JiraMovePanel = JiraMovePanel()

    init {
        DaggerJiraDIComponent.builder()
                .jiraModule(JiraModule(this, project))
                .build().inject(this)
        isModal = true
        presenter.load()
        init()
    }

    override fun createCenterPanel(): JComponent? = panel

    fun setIssue(issue: String) = panel.setIssue(issue)
}
```

如果现在运行插件，并在设置中使用我之前提到的正则表达式，您将看到，每当启动`JiraMoveAction`时，它都会根据当前分支将`issue`字段设置为`Jira`正确的`issue ID`。

![](https://raw.githubusercontent.com/qingmei2/qingmei2-blogs-art/master/ide-plugin/part4_6.png)

## 第七步：用Retrofit和RxJava请求网络

剩下唯一要做的事情就是为`Jira`的`issue`获取下一个`Transition`，并在用户按下`OK`按钮时移动`issue`。为此，我们将使用`Retrofit`和`RxJava`。

和往常一样，首先将新的依赖声明在`build.gradle`文件中。

```groovy
compile 'com.squareup.retrofit2:retrofit:2.5.0'
compile 'com.squareup.retrofit2:adapter-rxjava2:2.5.0'
compile 'com.squareup.retrofit2:converter-gson:2.5.0'
compile 'io.reactivex.rxjava2:rxjava:2.2.5'
compile 'com.github.akarnokd:rxjava2-swing:0.3.3'  // 译者注：注意这个compile
```

最后`compile`的依赖，可能会让你耳目一新，在`RxJava`中，我们需要一组`Scheduler`来进行订阅和观察。我们显然不能使用`Android`的，而是需要在事件分发线程或`EDT`中运行代码，新库将为我们提供`EDT`的调度程序。

我们将从编写`JiraService`开始，查看`Jira API`文档后，实现起来并不复杂：

```kotlin
interface JiraService {

    @GET("issue/{issueId}/transitions")
    fun getTransitions(@Header("Authorization") authKey: String,
                       @Path("issueId") issueId: String): Single<TransitionsResponse>

    @POST("issue/{issueId}/transitions")
    fun doTransition(@Header("Authorization") authKey: String,
                     @Path("issueId") issueId: String,
                     @Body transitionData: TransitionData): Completable
}
```

现在我们可以通过`Dagger`将`JiraService`的依赖向外暴露：

```kotlin
@Provides
fun providesJiraService(component: JiraComponent) : JiraService {
    val jiraURL = component.url
    val retrofit = Retrofit.Builder()
            .addConverterFactory(GsonConverterFactory.create())
            .addCallAdapterFactory(
                RxJava2CallAdapterFactory.create())
            .baseUrl(jiraURL)
            .build()

    return retrofit.create(JiraService::class.java)
}
```

在`Presenter`中，我们现在可以注入`JiraService`并将其与`RxJava`一起使用，以获取给定`issue`的`Transition`。

请注意，我们使用`SwingSchedulers.edt()`进行线程切换。代码非常简单，使用`Basic Auth`，我们将获得所有`Transition`的响应，然后将其传递到视图(`JiraMoveDialog`)，以将其添加到组合框。

如果发生`error`，我们在`view.error()`处理，它将显示带有错误详细信息的通知弹窗。

```kotlin
private fun getTransitions() {
    val auth = getAuthCode()
    disposable = jiraService.getTransitions(auth, issue)
            .subscribeOn(Schedulers.io())
            .observeOn(SwingSchedulers.edt())
            .subscribe(
                    { response ->
                        view.setTransitions(response.transitions)
                    },
                    { error ->
                        view.error(error)
                    }
            )
}

private fun getAuthCode() : String {
    val username = component.username
    val token = component.token
    val data: ByteArray =  
        "$username:$token".toByteArray(Charsets.UTF_8)
    return "Basic ${Base64.encode(data)}"
}
```

对话框的新方法可以设置`Transition`并显示`error`：

```kotlin
fun setTransitions(transitionList: List<Transition>) {
    for(transition in transitionList) {
        panel.addTransition(transition)
    }
}

fun error(throwable: Throwable) {
    val noti = NotificationGroup("myplugin",   
        NotificationDisplayType.BALLOON, true)
    noti.createNotification("Error", throwable.localizedMessage,
        NotificationType.ERROR, null).notify(project)
}
```

立即运行插件，完成需要的配置后，效果如下：

![](https://raw.githubusercontent.com/qingmei2/qingmei2-blogs-art/master/ide-plugin/part4_7.png)

## 第八步：执行 Transitions

最后一步，当用户按下`OK`按钮时，将`issue`移到组合框中显示的`Transition`中，`Presenter`中的代码再次使用`RxJava`，如下所示。

```kotlin
fun doTransition(selectedItem: Transition, issue: String) {
    val auth = getAuthCode()
    val transition = TransitionData(selectedItem)

    disposable = jiraService.doTransition(auth, issue, transition)
            .subscribeOn(Schedulers.io())
            .observeOn(SwingSchedulers.edt())
            .subscribe(
                    {
                        view.success()
                    },
                    { error ->
                        view.error(error)
                    }
            )
}
```

现在，在对话框中，我们必须重写`doOKAction()`以执行从`Presenter`传递过来的`Transition`，我们还创建了`success()`以关闭对话框并通知用户。

```kotlin
override fun doOKAction() =   
    presenter.doTransition(
        panel.getTransition(), panel.txtIssue.text
    )
fun success() {
    close(DialogWrapper.OK_EXIT_CODE)

    val noti = NotificationGroup("myplugin",    
        NotificationDisplayType.BALLOON, true)

    noti.createNotification("Success", "Issue moved",  
        NotificationType.INFORMATION, null).notify(project)
}
```

如果您现在运行插件，则无需离开`Android Studio`就可以进行`Jira`的更新。如果一切顺利，您现在应该会看到此弹出窗口，最重要的是，`Jira`中的`issue`现在应该移至下一阶段。

![](https://raw.githubusercontent.com/qingmei2/qingmei2-blogs-art/master/ide-plugin/part4_8.png)

这就是所有的内容了！如有需要，您现在可以进行更多扩展，并创建`Action`以使所有`issue`都处于`ToDo`状态、显示说明并在选择它们后使用正确的`issue id`创建新分支（或查看我的[Droidcon UK](https://skillsmatter.com/skillscasts/12166-write-your-own-android-studio-plugin-and-automate-everything)演讲以[获取代码](https://github.com/marcosholgado/demo-plugin)）。

请记住，本文的代码在该系列[GitHub Repo](https://github.com/marcosholgado/plugin-medium/tree/Part4)的`Part4`分支中可用。

在[下一篇文章](https://medium.com/@marcosholgado/write-an-android-studio-plugin-part-5-localization-and-notifications-cb036d867587)中，我们将整理一下代码，并创建一些`util`方法以在代码中复用。我们还将研究如何以比硬编码更好的方式存储字符串。

在此之前，如果您有任何问题，请随时发表评论或在[Twitter](https://www.twitter.com/orbycius)上关注我。


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
