## 动机

学习动机来源于[RxCache](https://github.com/VictorAlbertos/RxCache),在研究这个库的源码时，被这个库的设计思路吸引了，该库的原理就是通过动态代理和Dagger的依赖注入，实现Android移动端Retrofit的缓存功能。
 
 既然在项目中尝试使用这个库，当然要从设计的角度思考作者的思路，动态代理必然涉及到Java的反射，既然是反射，性能当然会有所降低，那么是否有更好的思路呢，使用动态代理的优势有哪些？

关于动态代理，百度上面的资料数不胜数，今天也借鉴其他前辈的学习总结，自己实践一次代理的实现。

## 静态代理
 
 代理分为动态代理和静态代理，我们先看静态代理的代码：

我们首先定义一个接口：

```java
public interface Subject {

    void enjoyMusic();

}
```

我们接下来实现一个Subject的实现类：

```java
public class RealSubject implements Subject {

    @Override
    public void enjoyMusic() {
        System.out.println("enjoyMusic");
    }
}
```
 在不考虑代理模式的情况下，我们调用Subject的真实对象，我们代码中必然是这样：

```
    @Test
    public void testNoProxy() throws Exception {
        Subject subject= new RealSubject();
        subject.enjoyMusic();
    }
```
 
 上面是我们的业务代码，我们这样使用当然没有问题，但是我们需要考虑的一点是，如果我们的业务代码中多次引用了这个类，并且在之后的版本迭代中，我们需要修改（或者替换）这个类，我们需要在引用这个对象的代码处进行修改——也就是说我们需要修改业务代码。
 
这显然不是良好的设计，我们希望业务代码不需要修改的前提下，进行RealSubject的修改（或者替换），这时我们可以通过代理模式，创建一个代理类，从而达到控制RealSubject对象的引用 ：

```
public class SubjectProxy implements Subject {
    private Subject subject = new RealSubject();
    @Override
    public void enjoyMusic() {
        subject.enjoyMusic();
    }
}
```

我们在业务代码中通过代理类，达到调用真实对象RealSubject的对应方法：
 
```
@Test
public void staticProxy() throws Exception {
    SubjectProxy proxy = new SubjectProxy();

    proxy.enjoyMusic();
}
```

这就是静态代理，优势是显然的，如果我们需要一个新的对象NewRealSubject代替RealSubject 应用在业务中，我们不需要修改业务代码，而是只需要在代理类中，将代码进行简单的替换：
```
private Subject subject = new RealSubject();//before

private Subject subject = new NewRealSubject();//after
```

同理，即使RealSubject类有所修改（比如说构造函数添加新的参数依赖），我们也不需要在每一处业务代码中添加一个新的参数，只需要在代理类中，对代理的真实对象进行简单修改即可。

## 瑕疵

现在我们看到了静态代理的优势，但是还有一点需要我们去思考，随着项目中业务量的逐渐庞大，真实对象类的功能可能越来越多：

```
//接口类
public interface Subject {

    void enjoyMusic();

    void enjoyFood();

    void enjoyBeer();
	
	//...甚至更多
}

//实现类
public class RealSubject implements Subject {

    @Override
    public void enjoyMusic() {
        System.out.println("enjoyMusic");
    }

    @Override
    public void enjoyFood() {
        System.out.println("enjoyFood");
    }

    @Override
    public void enjoyBeer() {
        System.out.println("enjoyBeer");
    }

	//...甚至更多
}
```

 这样岂不是说明，我们的代理类也要这样：

```
public class SubjectProxy implements Subject {

    private Subject subject = new RealSubject();

    @Override
    public void enjoyMusic() {
        subject.enjoyMusic();
    }

    @Override
    public void enjoyFood() {
        subject.enjoyFood();
    }

    @Override
    public void enjoyBeer() {
        subject.enjoyBeer();
    }
	
	//...甚至更多
}
```

 静态代理的话，确实如此，随着真实对象的功能增多，不可避免的，代理对象的代码也会随之臃肿，这是我们不希望看到的，我们更希望的是，即使真实对象的代码量再繁重，我们的代理类也不要有太多的改动和臃肿。

## 动态代理
 
 直接来看代码，我们首先声明一个接口和实现类：

```
public interface Subject {

    void enjoyMusic();

    void enjoyFood();

    void enjoyBeer();
}

public class RealSubject implements Subject {

    @Override
    public void enjoyMusic() {
        System.out.println("enjoyMusic");
    }

    @Override
    public void enjoyFood() {
        System.out.println("enjoyFood");
    }

    @Override
    public void enjoyBeer() {
        System.out.println("enjoyBeer");
    }
}
```
 这两位是老朋友了，接下来我们实现一个动态代理类：

```
public class DynamicProxy implements InvocationHandler {

    private Object subject;

    public DynamicProxy(Object subject) {
        this.subject = subject;
    }

    public Object bind() {
        return Proxy.newProxyInstance(subject.getClass().getClassLoader(),
                subject.getClass().getInterfaces(),
                this);
    }

    @Override
    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
        method.invoke(subject, args);
        return null;
    }
}
```
我们来看业务代码：

```
@Test
public void dynamicProxy() throws Exception {
    RealSubject realSubject = new RealSubject();
    Subject proxy = (Subject) new DynamicProxy(realSubject).bind();

    System.out.println(proxy.getClass().getName());

    proxy.enjoyMusic();
    proxy.enjoyFood();
    proxy.enjoyBeer();
}
```
动态代理中，我们可以看到，最重要的两个类：
Proxy 和 InvocationHandler
接下来分别对这两个类的功能进行描述.

###  InvocationHandler接口
我们先看一下Java的API文档：

> InvocationHandler is the interface implemented by the invocation handler of a proxy instance. 
Each proxy instance has an associated invocation handler. When a method is invoked on a proxy instance, the method invocation is encoded and dispatched to the invoke method of its invocation handler.

每一个动态代理类都必须要实现InvocationHandler这个接口，并且每个代理类的实例都关联到了一个handler，当我们通过代理对象调用一个方法的时候，这个方法的调用就会被转发为由InvocationHandler这个接口的 invoke 方法来进行调用。我们来看看InvocationHandler这个接口的唯一一个方法 invoke 方法：
 
 > Object invoke(Object proxy, Method method, Object[] args) throws Throwable

> proxy:　　指代最终生成的代理对象
> method:　　指代调用真实对象的某个方法的Method对象
> args:　　指代的是调用真实对象某个方法时接受的参数
 
 我们看到，我们的动态代理类中，实现了InvocationHandler接口的invoke方法，这里面三个参数的作用很简单，以proxy.enjoyMusic();为例，参数1表示最终生成的代理对象，参数2表示enjoyMusic()这个方法对象，参数3代表调用enjoyMusic()的参数，此例中是没有调用参数的。

有一个疑问是，这个proxy对象是什么呢，是这个new DynamicProxy(realSubject)吗？

当然不是，我们可以看到，这个代理对象proxy实际上是调用bind()方法获得的，也就是说是通过这个方法获得的：

> Proxy.newProxyInstance(subject.getClass().getClassLoader(),
                subject.getClass().getInterfaces(),
                this);
 
### Proxy类
 
 先看JavaAPI：

> Proxy provides static methods for creating dynamic proxy classes and instances, and it is also the superclass of all dynamic proxy classes created by those methods. 

Proxy这个类的作用就是用来动态创建一个代理对象的类，它提供了许多的方法，但是我们用的最多的就是 newProxyInstance 这个方法：


```
public static Object newProxyInstance(ClassLoader loader, Class<?>[] interfaces, InvocationHandler h) throws IllegalArgumentException
```
> Returns an instance of a proxy class for the specified interfaces that dispatches method invocations to the specified invocation handler.

> * loader:　　一个ClassLoader对象，定义了由哪个ClassLoader对象来对生成的代理对象进行加载

>* interfaces:　　一个Interface对象的数组，表示的是我将要给我需要代理的对象提供一组什么接口，如果我提供了一组接口给它，那么这个代理对象就宣称实现了该接口(多态)，这样我就能调用这组接口中的方法了

> * h:　　一个InvocationHandler对象，表示的是当我这个动态代理对象在调用方法的时候，会关联到哪一个InvocationHandler对象上


可以看到，Proxy这个类才会帮助我们生成相应的代理类，它是如何知道我们需要生成什么类的代理呢？

看第二个参数，我们把Subject.class作为参数传进去时，那么我这个代理对象就会实现了这个接口，这个时候我们当然可以将这个代理对象强制类型转化，因为这里的接口是Subject类型，所以就可以将其转化为Subject类型了。

我们看到，我在业务代码中对生成的proxy进行了打印：

```
System.out.println(proxy.getClass().getName());

//结果：
//$Proxy0
```

也就是说，通过 Proxy.newProxyInstance 创建的代理对象是在jvm运行时动态生成的一个对象，它并不是我们的InvocationHandler类型，也不是我们定义的那组接口的类型，而是在运行是动态生成的一个对象，并且命名方式都是这样的形式，以$开头，proxy为中，最后一个数字表示对象的标号。

到此，动态代理的基本知识就告一段落了。


## 参考资料:

java的动态代理机制详解:(对我帮助很大，衷心感谢！)
http://www.cnblogs.com/xiaoluo501395377/p/3383130.html

知乎:Java 动态代理作用是什么？
https://www.zhihu.com/question/20794107

本文sample代码已托管github：

https://github.com/qingmei2/Samples-Android/tree/master/SampleProxy/app/src/test/java/com/qingmei2/sampleproxy
