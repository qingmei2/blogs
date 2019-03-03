# [译]使用MVI打造响应式APP(一):到底是什么Model

> 原文：[《REACTIVE APPS WITH MODEL-VIEW-INTENT - PART1 - MODEL》](http://hannesdorfmann.com/android/mosby3-mvi-1)  
作者：[Hannes Dorfmann](http://hannesdorfmann.com)  
译者：[却把清梅嗅](https://github.com/qingmei2)  

有朝一日，我突然发现我对于`Model`层的定义 **全部是错误的**，更新了认知后，我发现曾经我在`Android`平台上主题讨论中的那些困惑或者头痛都消失了。

从结果上来说，最终我选择使用 `RxJava` 和 `Model-View-Intent(MVI)` 构建 **响应式的APP**，这是我从未有过的尝试——尽管在这之前我开发的APP也是响应式的，但 **响应式编程** 的体现与这次实践相比，完全无法相提并论，在接下来我将要讲述的一系列文章中，你也会感受到这些。但作为系列文章的开始，我想先阐述一个观点：

> **所谓的`Model`层到底是什么，我之前对`Model`层的定义出现了什么问题？**

我为什么说 **我对`Model`层有着错误的理解和使用方式** 呢？当然，现在有很多架构模式将`View`层和`Model`层进行了分离，至少在`Android`开发的领域，最著名的当属`Model-View-Controller (MVC)`、`Model-View-Presenter (MVP)`和`Model-View-ViewModel (MVVM)`——你注意到了吗？这些架构模式中，`Model`都是不可或缺的一环，但我意识到 **在绝大数情况下，我根本没有`Model`**。

举例来说，一个简单的从后端拉取`Person`列表情况下，传统的`MVP`实现方式应该是这样的：

```Java
class PersonsPresenter extends Presenter<PersonsView> {

  public void load(){
    getView().showLoading(true); // 展示一个 ProgressBar

    backend.loadPersons(new Callback(){
      public void onSuccess(List<Person> persons){
        getView().showPersons(persons); // 展示用户列表
      }

      public void onError(Throwable error){
        getView().showError(error); // 展示错误信息
      }
    });
  }
}
```

但是，这段代码中的`Model`到底是指什么呢？是指后台的网络请求吗？不，那只是业务逻辑。是指请求结果的用户列表吗？不，它和ProgressBar、错误信息的展示一样，仅仅只代表了`View`层所能展示内容的一小部分而已。

那么，`Model`层究竟是指什么呢？

从我个人理解来说,`Model`类应该定义成这样：

```Java
class PersonsModel {
  // 在真实的项目中，需要定义为私有的
  // 并且我们需要通过getter和setter来访问它们
  final boolean loading;
  final List<Person> persons;
  final Throwable error;

  public(boolean loading, List<Person> persons, Throwable error){
    this.loading = loading;
    this.persons = persons;
    this.error = error;
  }
}
```

这样的实现，`Presenter`层应该这样实现：

```Java
class PersonsPresenter extends Presenter<PersonsView> {

  public void load(){
    getView().render( new PersonsModel(true, null, null) ); // 展示一个 ProgressBar

    backend.loadPersons(new Callback(){
      public void onSuccess(List<Person> persons){
        getView().render( new PersonsModel(false, persons, null) );  // 展示用户列表
      }

      public void onError(Throwable error){
          getView().render( new PersonsModel(false, null, error) ); // 展示错误信息
      }
    });
  }
}
```

现在，`View`层持有了一个`Model`，并且能够借助它对屏幕上的控件进行`rendered`(渲染)。这并非什么新鲜的概念，`Trygve Reenskaug`在1979年时，其对最初版本的`MVC`定义中具有相似的概念：**`View`观察`Model`的变化**。


然而，`MVC`这个术语被用来描述太多种不同的模式，这些模式与`Reenskaug`在1979年制定的模式并不完全相同。比如后端开发人员使用`MVC`框架，iOS有`ViewController`，到了`Android`领域`MVC`又被如何定义了呢？`Activity`是`Controller`吗? 那这样的话`ClickListener`又算什么呢？如今，`MVC`这个术语变成了一个很大的误区，它错误地理解和使用了Reenskaug最初制定的内容——这个话题到此为止，再继续下去整个文章就会失控了。

言归正传，`Model`的持有将会解决许多我们在`Android`开发中经常遇到的问题：

* 1.状态问题
* 2.屏幕方向的改变
* 3.在页面堆栈中导航
* 4.进程终止
* 5.单向数据流的不变性
* 6.可调试和可重现的状态
* 7.可测试性

要讨论这些关键的问题，我们先来看看“传统”的`MVP`和`MVVM`的实现代码中如何处理它们，然后再谈`Model`如何跳过这些常见的陷阱。

### 1.状态问题

**响应式App**，这是最近非常流行的话题，不是吗？所谓的 **响应式App** 就是 **应用会根据状态的改变作出UI的响应**，这句话里有一个非常好的单词：**状态**。什么是状态呢？大多数时间里，我们将 **状态** 描述为我们在屏幕中看到的东西，例如当界面展示`ProgressBar`时的`loading state`。

很关键的一点是，我们前端开发人员倾向专注于UI。这不一定是坏事，因为一个好的UI体验决定了用户是否会用你的产品，从而决定了产品能否获得成功。但是看看上述的`MVP`示例代码（不是使用了`PersonModel`的那个例子），这里UI的状态由`Presenter`进行协调，`Presenter`负责告诉`View`层如何进行展示。

`MVVM`亦然，我想在本文中对`MVVM`的两种实现方式进行区分：第一种依赖`DataBinding`库，第二种则依赖`RxJava`；对于依赖`DataBinding`的前者，其状态被直接定义于`ViewModel`中：

```Java
class PersonsViewModel {
  ObservableBoolean loading;
  // 省略...

  public void load(){

    loading.set(true);

    backend.loadPersons(new Callback(){
      public void onSuccess(List<Person> persons){
      loading.set(false);
      // 省略其它代码，比如对persons进行渲染
      }

      public void onError(Throwable error){
        loading.set(false);
        // 省略其它代码，比如展示错误信息
      }
    });
  }
}
```

使用`RxJava`实现`MVVM`的方式中，其并不依赖`DataBinding`引擎，而是将`Observable`和UI的控件进行绑定，例如：

```Java
class RxPersonsViewModel {
  private PublishSubject<Boolean> loading;
  private PublishSubject<List<Person> persons;
  private PublishSubject loadPersonsCommand;

  public RxPersonsViewModel(){
    loadPersonsCommand.flatMap(ignored -> backend.loadPersons())
      .doOnSubscribe(ignored -> loading.onNext(true))
      .doOnTerminate(ignored -> loading.onNext(false))
      .subscribe(persons)
      // 实现方式并不惟一
  }

  // 在View层订阅它 (比如 Activity / Fragment)
  public Observable<Boolean> loading(){
    return loading;
  }

  // 在View层订阅它 (比如 Activity / Fragment)
  public Observable<List<Person>> persons(){
    return persons;
  }

  // 每当触发此操作 (即调用 onNext()) ，加载Persons数据
  public PublishSubject loadPersonsCommand(){
    return loadPersonsCommand;
  }
}
```

当然，这些代码并非完美，您的实现方式可能截然不同；我想说明的是，通常在`MVP`或者`MVVM`中，**状态** 是由`ViewModel`或者`Presenter`进行驱动的。

这导致下述情况的发生：

* 1.业务逻辑本身也拥有了状态，`Presenter`（或者`ViewModel`）本身也拥有了状态（并且，你还需要通过代码去同步它们的状态使其保持一致），同时，`View`可能也有自己的状态（比方说，调用`View`的`setVisibility()`方法设置其可见性，或者`Android`系统在重新创建时从`bundle`恢复状态）。  

* 2.`Presenter`（或`ViewModel`）有任意多个输入（`View`层触发行为并交给`Presenter`处理），这是ok的，但同时`Presenter`也有很多输出（或`MVP`中的输出通道，如`view.showLoading()`或`view.showError()`;在`MVVM`中，`ViewModel`的实现中也提供了多个`Observable`，这最终导致了`View`层，`Presenter`层和业务逻辑中状态的冲突，在处理多线程的时候，这种情况更明显。

在好的情况下，这只会导致视觉上的错误，例如同时显示加载指示符（“加载状态”）和错误指示符（“错误状态”），如下所示：

![](https://upload-images.jianshu.io/upload_images/2583346-2d6dca2b86eacf6b.gif?imageMogr2/auto-orient/strip%7CimageView2/2/w/218/format/webp)

在最糟糕的情况下，您从崩溃报告工具（如`Crashlytics`）接收到了一个严重的错误报告，但您无法重现这个错误，因此也几乎无从着手去修复它。

如果从 **底层** (业务逻辑层)到 **顶层** (UI视图层)，**有且仅有一个真实描述状态的源**，会怎么样呢？事实上，我们已经在文章的开头谈论`Model`的时候，就已经通过案例，把相似的概念展示了出来：

```Java
class PersonsModel {
  final boolean loading;
  final List<Person> persons;
  final Throwable error;

  public(boolean loading, List<Person> persons, Throwable error){
    this.loading = loading;
    this.persons = persons;
    this.error = error;
  }
}
```

你猜怎么了？ **Model映射了状态**，当我想通了这点，许多状态相关的问题迎刃而解（甚至在编码之前就已经被避免了）；现在`Presenter`层变得只有一个输出了：

> getView().render(PersonsModel)

它对应了一个数学上简单的函数，比如`f(x) = y`,对于多个输入的函数，对应的则是`f(a,b,c)`,但也是一个输出。

**并非对所有人来说数学都是香茗，就好像数学家并不清楚bug是什么——但软件工程师需要去品尝它。**

了解`Model`到底是什么以及如何建立对应的`Model`非常重要，因为最终`Model`可以解决 **状态问题**。

### 2.屏幕方向的改变

> 译者注：针对 **屏幕旋转后的状态回溯** 这个问题，已经可以通过Google官方发布的`ViewModel`组件进行处理，开发者不再需要为此烦恼，但本章节仍值得一读。

`Android`设备上的 **屏幕旋转** 是一个有足够挑战性的问题;忽视它是一个最简单的解决方案，即 **每次屏幕旋转，都对数据重新进行加载** 。这确实行之有效，大多数情况下，您的APP也在离线状态下工作，其数据来源于数据库或者其它本地缓存，这意味着屏幕旋转后的数据加载速度是很快的。

但是，个人而言我不喜欢看到加载框，哪怕加载速度是毫秒级别的，因为我认为这并非完美的用户体验，因此大家（包括我）开始使用`MVP`，这其中包括了 **保留性的Presenter**——这样就可以 **在屏幕旋转时分离和销毁View层**，而`Presenter`则会保存在内存中不会被销毁，然后`View`层会再次连接到`Presenter`。

使用`RxJava`的`MVVM`也可以实现相同的概念，但请牢记，一旦`View`对`ViewModel`取消了订阅，可观察的流就会被销毁，这个问题你可以用`Subject`解决；对于`DataBinding`构建的`MVVM`来讲，`ViewModel`由`DataBinding`直接绑定到`View`层，为了避免内存泄露，需要我们在屏幕旋转时及时销毁`ViewModel`。

对于 **保留性的Presenter** 或者 **ViewModel** 的问题是： 我们如何将`View`的状态在屏幕旋转之后回溯，保证`View`和`Presenter`再次回到之前相同的状态？我编写了一个名为 [Mosby](https://github.com/sockeqwe/mosby) 的MVP库，其包含一个名为`ViewState`的功能，它基本上将业务逻辑的状态与`View`同步。 [Moxy](https://github.com/Arello-Mobile/Moxy),另一个MVP库，提出了一个非常有趣的解决方案——通过使用`commands`在屏幕方向更改后重现View的状态：

![](https://upload-images.jianshu.io/upload_images/2583346-0eadd2ec67370541.gif?imageMogr2/auto-orient/strip%7CimageView2/2/w/959/format/webp)

针对`View`层状态的问题,我很确定还有其他的解决方案。让我们退后一步，归纳一下这些库试图解决的问题：那就是我们已经讨论过的 **状态问题**。

再次重申，我们通过一个 **能反映当前状态的Model** 和一个**渲染Model的方法** 解决了这个问题，就像调用`getView().render(PersonsModel)`一样简单。

### 3.在页面堆栈中导航

当`View`不再使用时，是否还有保留`Presenter`（或`ViewModel`）的必要？比如，用户跳转到了另外一个界面，这导致`Fragment`(`View`)被另外的`Fragment`给`replace`了，因此`Presenter`已经不在被任何`View`持有。

如果没有`View`层和`Presenter`进行关联，`Presenter`自然也无法根据业务逻辑，将最新的数据反映在`View`上。但如果用户又回来了怎么办（比如按下后退按钮），是 **重新加载数据** 还是 **重用现有的Presenter**?——这看起来像是一个哲学问题。

通常用户一旦回到之前的界面，他会期望回到之前的界面继续操作。这仍然像是第二小节关于`View`层 **状态恢复** 的问题，解决方案简明扼要：当用户返回时，我们得到 **代表状态的Model** ，然后只需调用 `getView().render(PersonsModel)` 对`View`层进行渲染。

### 4.进程终止

**进程终止是一件坏事，并且我们需要依赖一些库以帮助我们在进程终止后对状态进行恢复**——我认为这是`Android`开发中常见的一种误解。

首先，进程终止的原因只有一个，并且有足够充分的理由——`Android`操作系统需要更多资源用于其他应用程序或节省电池。**如果你的APP处于前台并且正在被用户主动使用时，这种情况永远不会发生**，因此，遵纪守法，不要与平台作斗争了（就是不要执拗于所谓的进程保活了）。如果你真的需要在后台进行一些长时间的工作，请使用`Service`,这也是向操作系统发出信号，告知您的App仍处于“主动使用状态”的 **唯一方式** 。

如果进程终止了，`Android`会提供一些回调以供 **保存状态**，比如`onSaveInstanceState()`——没错，又是 **状态** 。我们应该将`View`的信息保存在`Bundle`中吗？我们是否也应该把`Presenter`中的状态保存到`Bundle`中？那么业务逻辑的状态呢？又是老生常谈的问题，就和上面三个小节谈到的一样。

我们只需要一个代表整个状态的`Model`类，我们很容易将`Model`保存在`Bundle`中并在之后对它进行恢复。但是，我个人认为大部分情况下最好不保存状态，而是 **重新加载整个界面**，就像我们第一次启动App一样。 想想显示新闻列表的 `NewsReader App`。 当App被杀掉，我们保存了状态，6小时后用户重新打开App并恢复了状态，我们的App可能会显示过时的内容。因此,这种情况下，也许不存储`Model`和状态、而对数据重新加载才是更好的策略。

### 5.单向数据流的不变性

在这里我不打算讨论不变性(`immutabiliy`)的优势，因为有很多资源讨论这个问题。我们想要一个不可变的`Model`（代表状态）。为什么？因为我们想要唯一的状态源，在传递`Model`时，我们不希望App中的其他组件可以改变我们的`Model`或者`State`。

让我们假设编写一个简单的计数器App，它具有递增和递减的功能按钮，并在`TextView`中显示当前计数器值。 如果我们的`Model`（在这种情况下只是计数器值，即一个整数）是不可变的，那么我们如何更改计数器？

我很高兴被问到这个问题，按钮被点击时，我们并非直接操作`TextView`。我的建议是：

* 1.我们的`View`层应该有一个类似`view.render(...)`的方法；  
* 2.我们的`Model`是不可变的，因此不可直接修改`Model`;  
* 3.`View`的渲染有且只有一个来源:即业务逻辑。

我们将点击事件 **下沉** 到业务逻辑层。业务逻辑知道当前的`Model`（例如，持有一个私有的成员`Model`，它代表着当前的状态）, 这之后根据旧的Model，创建一个新的带有增量/减量值的`Model`。

![](http://hannesdorfmann.com/images/mvi-mosby3/counter.png)

这样我们建立了一个 **单向数据流**，业务逻辑作为单一源用于创建不可变的`Model`实例，但对于一个计数器来讲未免有点小题大做，不是吗？诚然，是的，计数器只是一个简单的应用程序。大多数应用程序都是以简单的应用程序开始，但复杂性增长很快——从我的角度来看，**单向数据流和不可变模型是必要的**，这会使简单的应用程序，在复杂性递增的同时，依然保持着简单（对开发者而言）。

### 6.可调试和可重现的状态

此外，**单向数据流保证了我们的应用程序易于调试**。下次我们从Crashlytics获得崩溃报告时，我们可以轻松地重现并修复此崩溃，因为所有必需的信息都已附加到崩溃报告中了。

什么叫做必需的信息？那就是当前的`Model`和用户用户在崩溃发生时想要执行的操作（比如，点击减量按钮）。这就是我们重现这次崩溃所需的全部信息，这些信息非常容易收集并附加在崩溃报告中。

如果没有**单项数据流**（比如，对`EventBus`的滥用，或者将`CounterModels`的私有域暴露出来），或者没有不变性（这会导致我们不知道谁实际更改了`Model`），那么bug的复现就没那么容易了。

### 7.可测试性

“传统”的`MVP`或`MVVM`提高了应用程序的可测试性。`MVC`也是可测试的：没有人说我们必须将所有业务逻辑放入`Activity`中。使用表示状态的`Model`，我们可以**简化单元测试的代码**，因为我们可以简单地检查`assertEqual(expectedModel，model)`。这使我们避免了许多必须要`Mock`的对象。

此外，这也减少了很多验证的测试，即某些方法是否被调用（比如`Mockito.verify(view, times(1)).showFoo()`），最终，这使得我们的单元测试代码更具可读性，易于理解并且易于维护，因为我们不必处理很多实际代码的实现细节。

## 总结

在这个博客文章系列的第一部分中，我们谈了很多关于理论的东西。我们真的需要关于专门讨论`Model`的博客吗？

我认为初步地理解`Model`的确很重要，这也有助于我们避免一些会遇到的问题。`Model`并不意味着业务逻辑，它是生成`Model`的业务逻辑（比如，一次交互，一个用例，一个仓库或者你在APP中调用的任何东西）。

在接下来的第二部分中，当我们最终使用`Model-View-Intent`构建一个**响应式App** 时，我们将看到`Model`的实际应用。演示的APP是一个虚构的在线商店的应用程序，敬请关注。

![](https://upload-images.jianshu.io/upload_images/2583346-7a75b9bf22c0f738.gif?imageMogr2/auto-orient/strip%7CimageView2/2/w/202/format/webp)


**--------------------------广告分割线------------------------------**

## 系列目录

> **《使用MVI打造响应式APP》系列目录**  
> * [[译]使用MVI打造响应式APP(一):Model层到底代表什么](https://github.com/qingmei2/android-programming-profile/blob/master/src/Android-MVI/%5B%E8%AF%91%5D%E4%BD%BF%E7%94%A8MVI%E6%89%93%E9%80%A0%E5%93%8D%E5%BA%94%E5%BC%8FAPP%5B%E4%B8%80%5D%3AModel%E5%B1%82%E5%88%B0%E5%BA%95%E4%BB%A3%E8%A1%A8%E4%BB%80%E4%B9%88.md)  
> * [[译]使用MVI打造响应式APP[二]:View层和Intent层](https://github.com/qingmei2/android-programming-profile/blob/master/src/Android-MVI/%5B%E8%AF%91%5D%E4%BD%BF%E7%94%A8MVI%E6%89%93%E9%80%A0%E5%93%8D%E5%BA%94%E5%BC%8FAPP%5B%E4%BA%8C%5D%3AView%E5%B1%82%E5%92%8CIntent%E5%B1%82.md)  
> * [[译]使用MVI打造响应式APP[三]:StateReducer](https://github.com/qingmei2/android-programming-profile/blob/master/src/Android-MVI/%5B%E8%AF%91%5D%E4%BD%BF%E7%94%A8MVI%E6%89%93%E9%80%A0%E5%93%8D%E5%BA%94%E5%BC%8FAPP%5B%E4%B8%89%5D%3AStateReducer.md)  
> * [[译]使用MVI打造响应式APP[四]:IndependentUIComponents](https://github.com/qingmei2/android-programming-profile/blob/master/src/Android-MVI/%5B%E8%AF%91%5D%E4%BD%BF%E7%94%A8MVI%E6%89%93%E9%80%A0%E5%93%8D%E5%BA%94%E5%BC%8FAPP%5B%E5%9B%9B%5D%3AIndependentUIComponents.md)  
> * [[译]使用MVI打造响应式APP[五]:DebuggingWithEase](https://github.com/qingmei2/android-programming-profile/blob/master/src/Android-MVI/%5B%E8%AF%91%5D%E4%BD%BF%E7%94%A8MVI%E6%89%93%E9%80%A0%E5%93%8D%E5%BA%94%E5%BC%8FAPP%5B%E4%BA%94%5D%3ADebuggingWithEase.md)
> * [[译]使用MVI打造响应式APP[六]:RestoringState](https://github.com/qingmei2/android-programming-profile/blob/master/src/Android-MVI/%5B%E8%AF%91%5D%E4%BD%BF%E7%94%A8MVI%E6%89%93%E9%80%A0%E5%93%8D%E5%BA%94%E5%BC%8FAPP%5B%E5%85%AD%5D%3ARestoringState.md)
> * [[译]使用MVI打造响应式APP[七]:Timing,SingleLiveEventProblem](https://github.com/qingmei2/android-programming-profile/blob/master/src/Android-MVI/%5B%E8%AF%91%5D%E4%BD%BF%E7%94%A8MVI%E6%89%93%E9%80%A0%E5%93%8D%E5%BA%94%E5%BC%8FAPP%5B%E4%B8%83%5D%3ATiming%2CSingleLiveEventProblem.md)
> * [[译]使用MVI打造响应式APP[八]:Navigation](https://github.com/qingmei2/android-programming-profile/blob/master/src/Android-MVI/%5B%E8%AF%91%5D%E4%BD%BF%E7%94%A8MVI%E6%89%93%E9%80%A0%E5%93%8D%E5%BA%94%E5%BC%8FAPP%5B%E5%85%AB%5D%3ANavigation.md)

## 参考

> [［译］为什么使用MVI模式（MVI编写响应式安卓APP入门系列第一部分MODEL](https://www.jianshu.com/p/f875e24acf95)

## 关于我

Hello，我是[却把清梅嗅](https://github.com/qingmei2)，如果您觉得文章对您有价值，欢迎 ❤️，也欢迎关注我的[博客](https://www.jianshu.com/u/df76f81fe3ff)或者[Github](https://github.com/qingmei2)。

如果您觉得文章还差了那么点东西，也请通过**关注**督促我写出更好的文章——万一哪天我进步了呢？

* [我的Android学习体系](https://github.com/qingmei2/android-programming-profile)
* [关于文章纠错](https://github.com/qingmei2/Programming-life/blob/master/error_collection.md)
* [关于知识付费](https://github.com/qingmei2/Programming-life/blob/master/appreciation.md)
