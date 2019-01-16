> 本篇文章已授权微信公众号 guolin_blog （郭霖）独家发布

## 前言

[RxImagePicker : 支持RxJava2响应式流、灵活可高度定制的Android图片选择器。](https://github.com/qingmei2/RxImagePicker)

这是我花费了数月闲暇时间从零开始写的一个库，在这期间，我学习到了很多，我想把自己的一些所得所感，以及这期间的一些思路，能够通过一篇文章的形式讲述出来，这就是本文的起源。

## 一.动机

在展开本文之前，我希望能够占用一些篇幅先自我回答三个问题：

> **1. 为什么要"重复"造轮子？**
> **2. 要实现一个怎么样的轮子？**
> **3. 和网上成熟的三方图片选择库相比，特点以及不足？**

接下来我要花费 **很大一部分的篇幅**，根据上述的问题，阐述缘由，我坚信，相对于一个库而言，**设计的初衷和思想更为重要** 。

### 为什么“重复”造轮子

**“Stop Trying to Reinvent the Wheel ( 不要重复造轮子 ) ”**,可能是每个程序员入行被告知的第一条准则。

我是一个Github的（伪）开源爱好者，在公司的Android项目遇到APP图片选择的功能需求时，我花时间研究了Github上最受欢迎的那些图片选择库，这些库都是由行业各前辈花费很大心血一点点写出来的，也经过很多的项目和时间的检验，一点点迭代过来，从某种程度上讲，我认为**这些库已经非常稳定并且成熟**。

我很快意识到了一个很**“严重”**的问题，这些库对 **图片选择功能** 的实现，其原理基本都是基于以下的思路实现的：

> * 开发者先配置好自己的需求，比如，图片最大可选的数量，是否可选择视频，或者是图片缩略图的加载引擎（GlideEngine/PicassoEngine）等等;
> * 通过某个方法，**定向**打开某个固定的Activity，这个Activity负责展示图片选择的UI;
> * 开发者在Activity的onActivityResult()方法中（或者其它的方式，比如回调方法），对选择结果的处理。

很合理的一种方式，但是对于我个人而言，还略显不够，我曾经在我的 [ 这篇文章 ](https://www.jianshu.com/p/c69b0e4e18f1)中阐述了我的一个观点：

> 事实上，现在网络上越来越多出现别人封装好的RecyclerViewAdapter或其他工具，很多都不可避免出现了 过度封装 的情况：它也许能够涵括大多数的需求，但是这也恰恰是它致命的弊端，在涉及一些新的功能时，它也许会突然无能为力——过度的封装带来了严重的耦合，这种问题是架构级的。**一个良好的设计需要更多的思考和尝试，更重要的也许是灵活，高度的可拓展性。**

试想这样一个场景，我在项目中集成了目前非常流行的 [ Matisse ](https://github.com/zhihu/Matisse),这是知乎开源出来的一个图片选择库，拥有着非常 **MaterialDesign** 的UI设计（这也是我无比坚定拥护Matisse的原因），我们可以看一下它的UI效果：

![Matisse.png](https://upload-images.jianshu.io/upload_images/7293029-e6be779eb52ee0c4.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

非常完美，在大多数项目上，Matisse 是这样的，但是当我尝试深入它的原理时，我有了一个困惑，那就是，我无法实现 **更私有化的UI定制**, 比如QQ这样的设计：

![qq.png](https://upload-images.jianshu.io/upload_images/7293029-7de5f731ec21ac72.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

不可否认，当我点击【相册】按钮时进入到的相册界面的UI，Matisse应该可以胜任（实际上，这也大概率需要修改Matisse的源码才能完全符合公司UI的设计），但是，在那之前，这个界面上的图片选择的UI需求，我需要自己去实现。

困惑就是，即使我实现了这个需求（简单分析来看，我需要自己实现一个Fragment来作为容器），**这个界面的图片选择和打开相册界面的图片选择，完全是不相干的两个功能**——它们都是图片选择，但一个是由我实现的，一个由借助三方的图片选择库实现，这是不合理的设计。

我更希望的是，能够有这样的一个架构，它能够将 **图片选择的【业务功能】和【UI设计】分离开** ，在稳定成熟的基础上，让开发者根据需求的变更，灵活的实现图片选择的功能—— 当我需要对图片选择的需求进行修改或者增加时，**我不需要替换框架，而是仅仅替换或者修改一个接口的实现。**

遗憾的是，我并没有找到这样的轮子，于是我在18年年初给自己设定了一个目标，**尝试设计出这样一个轮子**。

> 上文中，我以   [ Matisse ](https://github.com/zhihu/Matisse)为例进行了说明，但这并不是选择性地针对它，事实上， Matisse 是我认为非常优秀的库，同样，在我设计的框架中，  [ Matisse ](https://github.com/zhihu/Matisse)起到了至关重要的作用，这个后文会提到。

### 要实现一个怎么样的轮子

在我接触 [RxJava](https://github.com/ReactiveX/RxJava) 的这段时间里，这是一个需要一定学习成本的工具库，但我完全爱上了它，我希望我能实现的是一个 **支持RxJava响应式流、灵活且可高度定制的Android图片选择器**，因此我命名为 **RxImagePicker**。

#### 业务层
在业务层的设计上，它应该支持这些：

* Android 拍照
* Android 图片选择
* 以响应式数据流的格式返回数据（支持**Observable/Flowable/Single/Maybe**，对，就是RxJava）
* 动态配置响应式数据流的数据类型(就是返回值的类型，开发者不应该为它忧虑，而只需要调用一个接口):**File,Bitmap,或是Uri**
* 控制UI展示的逻辑，比如是直接打开一个Activity，还是作为一个Fragment被放入一个指定Id的容器中进行展示(就像QQ那样)。

我喜欢 [Retrofit](https://github.com/square/retrofit) 的设计 ,它将复杂的网络请求需求转换为一个接口进行配置，图片选择框架也许也可以这样做，比如这样：

```
public interface MyImagePicker {

    @Gallery    //打开相册选择图片
    @AsFile     //返回值为File类型
    Observable<File> openGallery();

    @Camera    //打开相机拍照
    @AsBitmap  //返回值为Bitmap类型
    Observable<Bitmap> openCamera();
}
```
接下来，我想要拍照或者图片选择，只需要通过注解配置好一个自定义的接口，然后通过代理实现即可：

```
//打开系统默认的图片选择器
private void onButtonClick() {
        new RxImagePicker.Builder()
                .with(this)
                .build()
                .create(MyImagePicker.class)   
                .openGallery()
                .subscribe(new Consumer<File>() {
                    @Override
                    public void accept(File file) throws Exception {
                        //做你想做的
                    }
                });
}
```

#### UI层

同时，它的UI层的设计上，我希望至少提供这些支持：

* 系统级别的拍照，相册选择图片功能
*  知乎主题图片选择器UI  （可选的，包括日间和夜间的两种主题）
* 微信主题图片选择器UI    （可选的）
* 自定义的图片选择器UI    （可选的）

下图就是库的sample所展示的效果：

##### 系统图片选择

![system.png](https://upload-images.jianshu.io/upload_images/7293029-21abd4d1e9d14a7e.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

##### 知乎主题

![zhihu.png](https://upload-images.jianshu.io/upload_images/7293029-b3fec6f3c44593a5.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

##### 微信主题

![wechat.png](https://upload-images.jianshu.io/upload_images/7293029-8f761e569db0999b.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

##### 展示到界面上

![result.png](https://upload-images.jianshu.io/upload_images/7293029-ae5c83926087c645.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


这其中的知乎主题的UI效果看起来和  [ Matisse ](https://github.com/zhihu/Matisse) 非常相似，事实上，UI层的架构就是基于   [ Matisse ](https://github.com/zhihu/Matisse) 进行的设计和修改——包括后面微信主题的图片选择UI，可以说，Matisse库的源码，是UI层的核心。


### 和其他库相比，差异在哪？

#### 1.灵活

上文花了巨幅的笔墨，说明了我为什么要设计一个这样的框架，无论是**UI层**还是**业务层**，一旦和目前项目的需求有了冲突（修改或者添加），开发者考虑的不应该是【这个库实现不了，干脆换一个库吧】或者【不管这个库，我去再单独实现一个】，而是，**基于同一个图片选择框架，修改或者添加对应配置的接口**。

#### 2.可定制

对于一个单独的依赖库，我添加了它的依赖，意味着我需要依赖它的所有。

如果 [RxImagePicker](https://github.com/qingmei2/RxImagePicker) 只提供了一个依赖，依赖它开发者可以随便使用知乎/微信的主题，也可以自定义UI，但是我无论选择哪一种实现方式，都意味着其他的主题没有用到，**这些没有用到的类和文件资源占用了apk的体积，站在开发者的角度而言也许问题不大，但对用户有限的流量来说，这是毁灭性的灾难。**

 [RxImagePicker](https://github.com/qingmei2/RxImagePicker)中， 类似于知乎/微信主题的UI，对于开发者是可选的：

```groovy
//下面的版本号是笔者写本文时最新的版本号，使用时，请以github上最新的版本作为参考

//【1】最基础的架构，仅提供了系统默认的图片选择器和拍照功能
compile 'com.github.qingmei2:rximagepicker:0.2.0'

//【2】提供了自定义UI图片选择器的基本组件，自定义UI的需求需要添加该依赖
compile 'com.github.qingmei2:rximagepicker_support:0.2.0'

//如果需要额外的UI支持，请选择依赖对应的UI拓展库
compile 'com.github.qingmei2:rximagepicker_support_zhihu:0.2.0'     //【3】知乎图片选择器
compile 'com.github.qingmei2:rximagepicker_support_wechat:0.2.0'    //【4】微信图片选择器
```

这样分层设计的原因是，开发者可以根据不同的需求选择添加不同的依赖：

* 当仅仅需要集成系统默认的功能时，可以添加最基础的依赖 *compile 'com.github.qingmei2:rximagepicker:0.2.0'* ，让自己项目中的图片选择功能能够通过RxJava的数据流观察到用户的行为。

* 当需要实现仿微信/知乎的图片选择功能时，可以添加对应主题的依赖，并进行对应的UI展示。

* 当需要实现私有化的UI定制时，开发者可以选择依赖 *compile 'com.github.qingmei2:rximagepicker_support:0.2.0'*, 其提供了最基础的UI组件（基于Matisse源码上，进行了一些修改以适应架构），以方便开发者进行私有化的UI设计，只需要实现RxImagePicker提供的**ICustomPickerView**接口，然后一切都不需要再管，交给RxImagePicker就行了。——事实上，【知乎】/【微信】主题图片选择器也是基于这些UI组件进行了UI的调整和封装。

其依赖结构为：

> 【3】知乎 /【4】微信 → 【2】UI support  →  【1】基础组件

![wechat_dependencies.png](https://upload-images.jianshu.io/upload_images/7293029-74e16b4737591be6.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

![support_dependencies.png](https://upload-images.jianshu.io/upload_images/7293029-35469d87c6813e60.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

![rxiamgepicker_dependencies.png](https://upload-images.jianshu.io/upload_images/7293029-aaaba056c32a9f46.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)



#### 3.需要考虑的成本

[RxImagePicker](https://github.com/qingmei2/RxImagePicker) 是一个支持RxJava2响应式流、灵活可高度定制的Android图片选择器，因此，添加它的依赖，默认就会自动添加 [RxJava](https://github.com/ReactiveX/RxJava) 和 [RxAndroid](https://github.com/ReactiveX/RxAndroid) 的依赖，因此，对于项目中并未使用RxJava的项目来说，所需要的成本是需要去认真考量的。

> 看到这里，如果您对于我个人的浅见有所认同，或者对这个库有一些兴趣，请不妨继续慢慢看下去，并尝试去使用它。

## 二. 在项目中使用它

本文**不阐述如何使用RxImagePicker**，因为它更应该详细地展示在项目的Github页面上。

[RxImagePicker](https://github.com/qingmei2/RxImagePicker) 项目的Github地址，内含各种UI主题（系统默认/微信/知乎）的使用Sample:

https://github.com/qingmei2/RxImagePicker

在阅读接下来的章节前，能够进入Github的项目主页，并花费一些时间下载尝试运行RxImagePicker的Sample也许是一个不错的选择。

我不能保证你会喜欢它，如果真的如此，花费时间阅读接下来的内容同样没有意义（虽然此时我还没有写接下来的内容，但我不认为它会比上面开头的内容少）。

## 三.从零实现RxImagePicker

### 3.1 业务层架构设计

灵感来源于  [Retrofit](https://github.com/square/retrofit) 和 [RxCache](https://github.com/VictorAlbertos/RxCache) 的设计 ,这两个库都通过将**复杂的【网络请求】/【数据缓存】的需求转换为一个接口进行配置**。

这种设计的优势很明显，开发者不需要真正了解底层的实现，通过 **注解** 和 **参数的注入** 对接口进行配置，同时通过动态代理完成对功能的实现。

于是在编码之前，我决定使用上述的这种方式实现 [RxImagePicker](https://github.com/qingmei2/RxImagePicker) 底部的搭建，开发者通过 **@Camera** 或者 **@Gallery** 注解来标记对应的行为（是相机拍照还是打开相册）。

同时，作为数据返回值的类型，我暂时提供了 **File/Bitmap/Uri** 三种可选项，其中Uri是默认的返回类型，开发者也不应该对返回值类型进行多余 的操作，最好是使用框架提供的接口，于是我添加了对应的注解  **@AsFile** , **@AsBitmap** 和 **@AsUri** 。

于是你能看到[RxImagePicker](https://github.com/qingmei2/RxImagePicker) 中自定义接口标准的使用方式：

```
public interface MyImagePicker {

    @Gallery    //打开相册选择图片
    @AsFile     //返回值为File类型
    Observable<File> openGallery();

    @Camera    //打开相机拍照
    @AsBitmap  //返回值为Bitmap类型
    Observable<Bitmap> openCamera();
}
```

同时，除了Observable，库的本身还提供了对RxJava其他响应式类型的支持，包括**Observable/Flowable/Single/Maybe**。

### 3.2 实现业务层

#### 使用动态代理

 研究过 [Retrofit](https://github.com/square/retrofit) 或者 [RxCache](https://github.com/VictorAlbertos/RxCache) 的源码的同学都知道，其底部原理都是基于Java的动态代理，通过反射的方式解析接口方法中的配置（比如参数/返回值/注解），生成对应的proxy对象。

[RxImagePicker](https://github.com/qingmei2/RxImagePicker)  也不例外，我仿照 Retrofit 和 RxCache,通过实现一个InvocationHandler的实例，负责管理proxy对象的生成，并在invoke（）方法中对接口方法解析，并将返回值返回。

```Java
public final class ProxyProviders implements InvocationHandler {
    //......
    //省略其他代码
    @Override
    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {

        return Observable.defer(new Callable<ObservableSource<?>>() {
            @Override
            public ObservableSource<?> call() throws Exception {
                //解析接口方法，获取配置的Bean对象
                ImagePickerConfigProvider configProvider = proxyTranslator.processMethod(method, args);
             
               //实例化UI组件的Projector，display()方法意味着展示UI
                proxyTranslator.instanceProjector(configProvider, fragmentActivity)
                        .display(customPickConfigurations);

                //这个返回值就是用户选择结果的流
                Observable<?> observable = rxImagePickerProcessor.process(configProvider);

                //根据返回值类型，进行数据转换
                Class<?> methodType = method.getReturnType();

                if (methodType == Observable.class) return Observable.just(observable);

                if (methodType == Single.class)
                    return Observable.just(Single.fromObservable(observable));

                if (methodType == Maybe.class)
                    return Observable.just(Maybe.fromSingle(Single.fromObservable(observable)));

                if (methodType == Flowable.class)
                    return Observable.just(observable.toFlowable(BackpressureStrategy.MISSING));

                throw new RuntimeException(method.getName() + " needs to return one of the next reactive types: observable, single, maybe or flowable");
            }
        }).blockingFirst();
    }
}
```

最终所有配置都会汇聚在一个实例化的 **ImagePickerConfigProvider** Bean对象中，这个类包含所有的个性化配置：

```
/**
 * Entity class for user config.
 */
public final class ImagePickerConfigProvider {

    private final String viewKey;
    private final boolean singleActivity;  //是否作为一个activity打开
    private final Class<? extends Activity> activityClass;

    private final SourcesFrom sourcesFrom;//是打开相册还是拍照
    private final ObserverAs observerAs;  //返回值的类型，File,Uri还是Bitmap

    private final int containerViewId;  //若作为一个View(比如Fragment)，承载它容器的Id
    private final ICustomPickerView pickerView;  //View组件的实例（比如Fragment）

    public ImagePickerConfigProvider(boolean singleActivity,
                                     String viewKey,
                                     SourcesFrom sourcesFrom,
                                     ObserverAs observerAs,
                                     ICustomPickerView pickerView,
                                     @IdRes int containerViewId,
                                     Class<? extends Activity> activityClass
    ) {
        this.sourcesFrom = sourcesFrom;
        this.observerAs = observerAs;
        this.pickerView = pickerView;
        this.containerViewId = containerViewId;
        this.viewKey = viewKey;
        this.singleActivity = singleActivity;
        this.activityClass = activityClass;
    }
    //省略get()方法
}
```
这样的好处显而易见，汇聚不同地方的配置，最终生成一个对象，它的职责是单一的，它保证了：在实现业务功能时，只需要持有一个对象，根据对应的属性进行对应的配置。

> 在这里，我们似乎并没有看到一些常见的配置，比如可选择图片的最大数量，UI的主题等等。原因是，这个对象只是管理业务功能的配置，这些UI的配置应该交给对应的UI层去存储并管理。

#### 使用Dagger2依赖注入

我希望我能够对自己库所 **额外添加的依赖，管理更加严苛一些**，在最基础的组件中，我甚至没有添加 [Glide](https://github.com/bumptech/glide) 或者 [picasso](https://github.com/square/picasso) ，因为我认为它们更应该在UI层的拓展library中被添加或者管理，这样能够进一步减少基础业务组件的体积。

使用Dagger2是一个艰难的决定，大家都知道，一个库的设计，依赖越少越好，这样就不会因为其他依赖库的升级，而导致库本身的bug。

> 比如说，我的一个库底层依赖了Glide，那么Glide在大版本升级时，API的改变有可能影响到我自己库的使用，这是很难避免的。

同时，**依赖的越多，意味着库本身的体积增加**，这本身就是一种局限性。

但我还是选择了Dagger2，因为在我看来这样利大于弊，因此，在选择使用 RxImagePicker 的时候，默认除了  [RxJava](https://github.com/ReactiveX/RxJava) 和 [RxAndroid](https://github.com/ReactiveX/RxAndroid) 的依赖,还额外依赖了 [dagger](https://github.com/google/dagger)——这是 [google](https://github.com/google) 基于 [square](https://github.com/square) 的dagger上，fork并自行拓展的**依赖注入库**。

dagger的使用让我对架构中配置的管理游刃有余，这也要感谢  [VictorAlbertos](https://github.com/VictorAlbertos) ，通过他的 RxCache ，让我知道了，依赖注入还可以这样用。

到这里，库本身的所有额外依赖都已经写清楚了：

![image.png](https://upload-images.jianshu.io/upload_images/7293029-ab27a68850dfa9fa.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

当然，对于项目中使用到了dagger和RxJava的同行来说，这种行为几乎没有影响。

#### 图片选择功能的实现

现在，通过动态代理和Dagger，我已经获取到了接口方法对应的所有配置，接下来就是要将UI和业务进行绑定了。

和 Retrofit 略微不同的是，前者并未涉及UI，因此我还需要寻求其它库的一些帮助。

[MLSDev：RxImagePicker](https://github.com/MLSDev/RxImagePicker)，感谢这个库，让我看到了曙光。

这是和我名字相同的一个库（实际上我的基础组件的部分设计参考了它，给我带来了很大的帮助，衷心感谢！），它也是用来提供Android设备上图片选择的功能，但是有一个缺陷，就是它使用的是系统默认的拍照和相册功能，如果你想要自定义UI，你需要拉他的源码进行修改，并自己实现UI。

虽然有局限性，但是它的原理是，创建一个不可见的Fragment，通过这个Fragment，控制Intent打开系统的相册或者相机，在onActiviyResult()方法中将数据交给Subject发射，开发者就能在代码中通过subscribe（）订阅获取对应的数据流了。

除此之外，我还研究参考了其它一些RxJava的拓展库，比如：

[**RxLifecycle**: Lifecycle handling APIs for Android apps using RxJava](https://github.com/trello/RxLifecycle)
[**RxPermissions**: Android runtime permissions powered by RxJava2](https://github.com/tbruyelle/RxPermissions)

他们的原理都很相似，通过在生命周期组件中添加一个Subject，用来接受并发射数据，然后将Subject返回给开发者，开发者订阅后，就能获取对应的数据。

我也实现了一个Fragment：

```
//略微进行了调整，并省略大量代码，这里仅仅阐述自己的思路
public final class SystemGalleryPickerView extends Fragment 
                  implements ICustomPickerView {

       //这个Subject发射的数据就是用户选择的图片
       private PublishSubject<Uri> publishSubject;
       //是否要终止本次订阅
       private PublishSubject<Integer> canceledSubject;
       
       //当用户调用接口的方法打开相册或者拍照时，代理对象实际会调用该方法
       //并将publishSubject强转成Obervable<T>返回给开发者
       //同时规定，当canceledSubject发射数据时，publishSubject停止订阅
       public Observable <Uri> pickImage() {
          publishSubject = PublishSubject.create();
          canceledSubject = PublishSubject.create();

          requestPickImage();
          return publishSubject.takeUntil(canceledSubject);
      }
        
        //实际上就是这个隐藏的Fragment打开相册，然后再onActivityResult()中监听
        //当用户选择的图片返回，将图片的Uri从intent中取出并交给PublishSubject发射
       @Override
    public void onActivityResult(int requestCode, int resultCode, Intent data) {
        if (resultCode == RESULT_OK) {
            if (publishSubject != null) {
            publishSubject.onNext(getActivityResultUri(data));
            publishSubject.onComplete();
            }
        } else {
            canceledSubject.onNext(requestCode);
        }
    }
}
```

#### 业务层和UI层的隔离尝试:ICustomPickerView

上文中，我的Fragment实现了ICustomPickerView，这是一个接口，用于定义UI层的一些行为：

```Java
public interface ICustomPickerView {
    //展示UI界面
    void display(FragmentActivity fragmentActivity,
                 @IdRes int viewContainer,
                 String tag,
                 ICustomPickerConfiguration configuration);
    //要返回的数据
    Observable<Uri> pickImage();
}
```
它的意义是，将用户的图片选择或者拍照行为，所需要展示的界面，抽象成一个接口，供RxImagePicker调用。

很好理解，我依赖了知乎主题的图片选择器，其中的**ZhihuImagePickerFragment**类负责展示知乎主题的UI：

```groovy
compile 'com.github.qingmei2:rximagepicker_support_zhihu:0.2.0' 
```

实际上，因为依赖关系是RxImagePicker被依赖于知乎主题的UI库，底层的RxImagePicker基础组件无法获取上层的 **ZhihuImagePickerFragment**类的引用，因此我设计了一个接口，就是**ICustomPickerView**。

这之后，再让ZhihuImagePickerFragment实现ICustomPickerView接口：

```Java
public class ZhihuImagePickerFragment extends Fragment implements
        IGalleryCustomPickerView {
        //...
}
```

这样，底层的组件只需要负责调用两个方法，**什么时候展示UI，以及什么时候返回数据**，至于是如何实现的，底层组件并不关心。

### 3.3 UI层的设计和实现

上文中，我已经定义好了UI层接口，对于一些拓展的UI选择器，我只需要按照UI层接口的规范，实现对应的逻辑即可。

UI层的设计我借鉴了   [ Matisse ](https://github.com/zhihu/Matisse) ，它的设计已经足够好，稳定性也一定比我自己实现好得多，拓展性功能的API也已经涵括了大多数的需求。

我花了一些时间基于 [ Matisse ](https://github.com/zhihu/Matisse) 的源码进行了部分的修改。

我把 [ Matisse ](https://github.com/zhihu/Matisse) 的代码分成了两层：

```groovy
//提供了自定义UI图片选择器的基本组件，自定义UI的需求需要添加该依赖
compile 'com.github.qingmei2:rximagepicker_support:0.2.0'

compile 'com.github.qingmei2:rximagepicker_support_zhihu:0.2.0'     //知乎图片选择器
```
其中我将公共的放入了 rximagepicker_support包中，作为UI层的基本组件，而详细的UI实现界面，我放入了rximagepicker_support_zhihu包中：

![image.png](https://upload-images.jianshu.io/upload_images/7293029-0b9d576e77d0154a.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


![image.png](https://upload-images.jianshu.io/upload_images/7293029-003ffd4523340d88.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

这之后就是长时间的UI界面代码的修改和编写，相比之前热情饱满的业务设计，UI层代码写起来真的有点枯燥（其间我加深了关于自定义控件的理解，收获颇多，但是我还是要说，很枯燥...）。

最终，除了zhihu主题的图片选择器，我还基于 Matisse 修改后的 UI组件，设计出了Wechat主题的图片选择器以供参考——就像文章开篇时的 sample 展示效果一样。

## 总结

经过几个月的学习和尝试，我最终实现出来了 [RxImagePicker](https://github.com/qingmei2/RxImagePicker)  ，坦白来讲，其中的设计并非如我预期的那般完美，但是它依然是我目前比较满意的作品。它达到了我想要的样子：**灵活** 且 **可高度定制** ，并且支持 **RxJava** 。

这不是终点，因为一个工具是依靠不断的尝试和迭代才能慢慢完善的，我已经在公司的项目中尝试使用了它，至少目前为止，它正在经历Production级别的检验。

这几个月的经历，RxImagePicker并不是一个很重的库，它仅仅是一个工具，便让我更加体会到了开源的艰难，而这才仅仅开始。

在此要特别感谢一位前辈，[ JessYan ](https://github.com/JessYanCoding),有幸和他在数次交流中，让我深刻理解了开源的意义，开源重要的不仅仅是 **方便快速实现** 和 **便于装X** ，更重要的是 **思想的交流** 和 **责任感**。

**我会坚持维护下去，也希望有朋友能够尝试使用它，并在 [issues](https://github.com/qingmei2/RxImagePicker/issues) 中提供反馈，我将第一时间进行回复。**

最后留下RxImagePicker的Github地址:

[RxImagePicker : 支持RxJava2响应式流、灵活可高度定制的Android图片选择器。](https://github.com/qingmei2/RxImagePicker)
https://github.com/qingmei2/RxImagePicker
