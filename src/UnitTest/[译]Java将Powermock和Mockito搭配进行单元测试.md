> 本文为Powermock官方文档Mockito篇的中文翻译
> 原文：https://github.com/powermock/powermock/wiki/Mockito
> 翻译：[却把清梅嗅](https://github.com/qingmei2)

 **笔者的Android单元测试相关系列：**
>[ Android单元测试：Mockito使用详解](http://blog.csdn.net/mq2553299/article/details/77014651)
[Android单元测试：使用本地数据测试Retrofit](http://blog.csdn.net/mq2553299/article/details/78471134)
[Android单元测试：测试RxJava的同步及异步操作](http://blog.csdn.net/mq2553299/article/details/78525488)
[Java 将Powermock和Mockito搭配进行单元测试](https://blog.csdn.net/mq2553299/article/details/81116676)
[Android 自动化测试 Espresso篇：简介&基础使用](http://blog.csdn.net/mq2553299/article/details/74067002)
[Android 自动化测试 Espresso篇：异步代码测试](http://blog.csdn.net/mq2553299/article/details/74490718)

## 简介

Powermock提供了基础的PowerMockito类，你仍然可以通过初始化 mock/object/class 并配置它们的校验、期望行为、或者其他，以达到通过Mockito配置和验证你的预期（例如times(), anyInt()）的目的。

所有的操作都需要再Class层级上配置 @RunWith(PowerMockRunner.class) 和 @PrepareForTest 注解

## 版本支持

![image.png](https://upload-images.jianshu.io/upload_images/7293029-0c1785b2144becba.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

## 用法
* * *

在下面的示例中，我们不对Mockito或PowerMockito API中的方法使用静态导入的方式，这便于我们观察实际代码是如何被测试的。 但是，我们强烈建议您静态导入方法的方式，以提高可读性。

请注意：Mockito团队在2.1.0的版本中增加了mock final 类型Class/Methods 的能力。 PowerMock 1.7.0版本之后支持此功能（请使用Mockito 2.8.9进行测试）。 使用PowerMock配置可以启用该功能。 如果您使用Mockito 2，建议使用Mockito来mock final 类型Class/Methods。

### Mock 静态方法

下面的代码将展示如何Mock以及打桩（stub）：

1.添加 `@PrepareForTest` 注解在你的Class层级上。

  ```java
  @PrepareForTest(Static.class) // Static.class contains static methods
  ```

2.调用`PowerMockito.mockStatic()` 以mock静态类（使用`PowerMockito.spy(class)` 来mock制定的方法）.

  ```java
  PowerMockito.mockStatic(Static.class);
  ```

3.使用Mockito.when()配置你的期望：

  ```java
  Mockito.when(Static.firstStaticMethod(param)).thenReturn(value);
  ```

注意：如果需要模拟java系统/ bootstrap类加载器（在java.lang或java.net等中定义的类）加载的类，则需要调用[this]（Mock-System）方法。

#### 如何验证行为 ####

想要验证静态方法是否被执行完毕，通过以下两个步骤：

   1.首先，调用 `PowerMockito.verifyStatic(Static.class) ` 来开始验证行为；
   2.然后调用 `Static.class` 的静态方法进行验证。

例如：

```java
PowerMockito.verifyStatic(Static.class); // 1
Static.firstStaticMethod(param); // 2
```

**重要**：对于每一个方法的验证，你都需要去调用 `verifyStatic(Static.class)`方法。

#### 如何使用参数匹配器 ####

Mockito的匹配器依然被应用在了PowerMock 的mock行为中，下面的代码讲述如何为每个被mock的静态方法使用自定义的参数匹配器：

```java
PowerMockito.verifyStatic(Static.class);
Static.thirdStaticMethod(Mockito.anyInt());
```
#### 如何验证方法被调用的准确次数 ####

你仍然可以使用Mockito.VerificationMode (e.g Mockito.times(x)) 通过`PowerMockito.verifyStatic(Static.class, Mockito.times(2))`的方式验证。

```java
PowerMockito.verifyStatic(Static.class, Mockito.times(1));
```

#### 如何为返回值为Void类型的静态方法抛出异常 ####

如果不是Private修饰的方法，则：

```java
PowerMockito.doThrow(new ArrayStoreException("Mock error")).when(StaticService.class);
StaticService.executeMethod();
```

注意，您可以对final修饰的 **类和方法** 执行相同的操作:

```java
PowerMockito.doThrow(new ArrayStoreException("Mock error")).when(myFinalMock).myFinalMethod();
```

对于Private的方法，请使用PowerMockito.when()，比如:

```java
when(tested, "methodToExpect", argument).thenReturn(myReturnValue);
```

#### 一个完整的案例，讲述了如何mock、打桩以及验证静态方法 ####

```java
@RunWith(PowerMockRunner.class)
@PrepareForTest(Static.class)
public class YourTestCase {
    @Test
    public void testMethodThatCallsStaticMethod() {
        // mock all the static methods in a class called "Static"
        PowerMockito.mockStatic(Static.class);
        // use Mockito to set up your expectation
        Mockito.when(Static.firstStaticMethod(param)).thenReturn(value);
        Mockito.when(Static.secondStaticMethod()).thenReturn(123);

        // execute your test
        classCallStaticMethodObj.execute();

        // Different from Mockito, always use PowerMockito.verifyStatic(Class) first
        // to start verifying behavior
        PowerMockito.verifyStatic(Static.class, Mockito.times(2));
        // IMPORTANT:  Call the static method you want to verify
        Static.firstStaticMethod(param);


        // IMPORTANT: You need to call verifyStatic(Class) per method verification, 
        // so call verifyStatic(Class) again
        PowerMockito.verifyStatic(Static.class); // default times is once
        // Again call the static method which is being verified 
        Static.secondStaticMethod();

        // Again, remember to call verifyStatic(Class)
        PowerMockito.verifyStatic(Static.class, Mockito.never());
        // And again call the static method. 
        Static.thirdStaticMethod();
    }
}
```

### 局部Mock

您可以通过使用PowerMockito.spy来局部mock一个方法，请注意（以下内容取自Mockito文档并同样适用于PowerMockito）：

**有时，你并不能使用默认的 `when(..)`方法为spies打桩，例如：**

```java
List list = new LinkedList();
List spy = spy(list);
//Impossible: real method is called so spy.get(0) throws IndexOutOfBoundsException (the list is yet empty)
when(spy.get(0)).thenReturn("foo");

//You have to use doReturn() for stubbing
doReturn("foo").when(spy).get(0);
```

### 如何验证行为

你只需要使用 ` Mockito.vertify() ` 进行默认的校验：

```java
Mockito.verify(mockObj, times(2)).methodToMock();
```

### 如何校验private的行为

使用 `PowerMockito.verifyPrivate()`, 比如：

```java
verifyPrivate(tested).invoke("privateMethodName", argument1);
```

这种方式同样适用于private 的静态方法。

### 如何对对象的创建进行mock

使用 `PowerMockito.whenNew()`, 比如：

```java
whenNew(MyClass.class).withNoArguments().thenThrow(new IOException("error message"));
```

请注意，您必须准备类_creating_用于测试的新MyClass实例，而不是`MyClass`本身。 (翻译无能，请参考下面的举例...)

比如说，如果执行`new MyClass()`的类被称为X，那么你必须执行`@PrepareForTest(X.class)`以使`whenNew`工作：

```java
@RunWith(PowerMockRunner.class)
@PrepareForTest(X.class)
public class XTest {
        @Test
        public void test() {
                whenNew(MyClass.class).withNoArguments().thenThrow(new IOException("error message"));

                X x = new X();
                x.y(); // y is the method doing "new MyClass()"
               
                ..
        }
}
```

### 如何验证新对象的实例化

使用 `PowerMockito.verifyNew` :

```java
verifyNew(MyClass.class).withNoArguments();
```

### 如何使用参数匹配器

Mockito的匹配器依然被应用在了PowerMock中：

```java
Mockito.verify(mockObj).methodToMock(Mockito.anyInt());  
```
> 译者注: 以参数匹配器为例，官方文档似乎重复讲了两次，实际上，这两处分别针对静态方法和普通的方法进行了分开的阐述，对于静态方法，我们只能通过PowerMock的API进行校验，但是对于普通的成员方法，使用Mockito提供的API即可。

### 一个完整讲述了Spy的案例：

```java
@RunWith(PowerMockRunner.class)
// We prepare PartialMockClass for test because it's final or we need to mock private or static methods
@PrepareForTest(PartialMockClass.class)
public class YourTestCase {
    @Test
    public void spyingWithPowerMock() {        
        PartialMockClass classUnderTest = PowerMockito.spy(new PartialMockClass());

        // use Mockito to set up your expectation
        Mockito.when(classUnderTest.methodToMock()).thenReturn(value);

        // execute your test
        classUnderTest.execute();

        // Use Mockito.verify() to verify result
        Mockito.verify(mockObj, times(2)).methodToMock();
    }
}
```

### 一个完整讲述了对Private方法的局部Mock的案例（PowerMock 1.3.6+版本后提供的支持）：


```java
@RunWith(PowerMockRunner.class)
// We prepare PartialMockClass for test because it's final or we need to mock private or static methods
@PrepareForTest(PartialMockClass.class)
public class YourTestCase {
    @Test
    public void privatePartialMockingWithPowerMock() {        
        PartialMockClass classUnderTest = PowerMockito.spy(new PartialMockClass());

        // use PowerMockito to set up your expectation
        PowerMockito.doReturn(value).when(classUnderTest, "methodToMock", "parameter1");

        // execute your test
        classUnderTest.execute();

        // Use PowerMockito.verify() to verify result
        PowerMockito.verifyPrivate(classUnderTest, times(2)).invoke("methodToMock", "parameter1");
    }
}
```

## 更多信息

想了解更多，请查看GitHub中的[source](https://github.com/powermock/powermock/tree/master/tests/mockito/junit4/src/test/java/samples/powermockito/junit4)。

另请阅读[Jayway团队博客](http://blog.jayway.com/)中的PowerMockito相关[博客](http://blog.jayway.com/2009/10/28/untestable-code-with-mockito-and-powermock/）。

## 关于译者

[却把清梅嗅](https://github.com/qingmei2)
Github:https://github.com/qingmei2
CSDN:https://blog.csdn.net/mq2553299
简书：https://www.jianshu.com/u/df76f81fe3ff


