# [译]使用MVI打造响应式APP(一):到底是什么Model

> 原文：[REACTIVE APPS WITH MODEL-VIEW-INTENT - PART1 - MODEL](http://hannesdorfmann.com/android/mosby3-mvi-1)  
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

**响应式App**，这是最近非常流行的话题，不是吗？
