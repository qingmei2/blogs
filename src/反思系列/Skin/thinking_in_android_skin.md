# 反思 | 开启B站少女心模式，探究APP换肤机制的设计与实现

## 概述

**换肤功能** 并非奇技淫巧，而是已非常普及，尤其当`Android Q`推出了 **深色模式** 之后，国内绝大多数主流应用都至少提供了 **日间** 和 **夜间** 两种模式。

对于无感的用户而言，这个功能实属鸡肋，但从另外一个角度上来说，这也是产品在雕琢 **用户极致体验** 过程中的一次尝试，为不同情景下，不同偏好的用户提供更多的选择性。

以 **哔哩哔哩** 为例，除了提供以上两种主题之外，还免费提供了充满 **少女心** 的粉色主题：

![](bilibili.png)

从产品的前瞻性上来看，国内在换肤功能的探索较国外是领先的，抽象的来看待`Android Q`的 **深色模式**，也无非是新增一种主题罢了，因此，开发者应该将角度放在更高的层级上：**为产品提供一套完善的换肤方案**，而非仅仅是  **适配深色模式** 。

想清楚这一点，开发者就不会将目光仅局限于技术本身——对于整个换肤体系而言，涵盖了UI、产品、开发、测试、运维等多名角色不同的关注点，而这些关注点最终却都依赖研发协助做决策，举例如下：

* **UI**：定义不同的UI组件不同的颜色属性，这些属性最终在不同的主题下，代表不同的颜色（日间模式下标题是黑色，但是夜间模式下，标题应该是白色）。
* **产品**：定义换肤功能的业务流程，从简单的换肤主页，换肤的交互，到不同主题下的不同展示、付费策略等等。
* **开发**：提供换肤功能的研发能力。
* **测试**：保证换肤功能的稳定性，比如自动化测试和便捷取色工具。
* **运维**：保证线上问题的快速定位和及时解决。

除此之外，还有更多可以深入思考的技术点，比如，随着主题越来越多，势必导致`APK`包体积的增大，是否有必要引入远程动态加载（`download & install`）的能力？借助不同角色的视角，我们可以提前规划好远景，接下来的编码也就更加遂心应手。

本文将针对 `Android` 应用整个换肤体系进行概括性的描述，读者应抛开对 **代码实现的细节** 的执著，从不同角色的需求去思考，**窥一斑而知全豹** ，为产品打造出健壮有力的技术支撑。

## 一、定义UI规范

换肤规范的目的是什么？对于UI设计和开发人员而言，设计与开发都应该基于统一且完整的规范之上进行，以掘金APP为例：

![](skin_juejin_design.png)

对于UI设计人员，在APP不同的主题下，控件的颜色不再是一个单一的值，而应该用一个通用的`key`来进行定义，如上图所示，「标题」的颜色，在日间应该是黑色`#000000`，而深色模式下则应该为白色`#FFFFFF`，同理，「次级标题」、「主背景色」、「分割线颜色」，都应该随着不同的主题下，对应不同的值。

设计人员在设计时，仅需要针对页面每一个元素填充好对应的`key`，根据规范很清晰地完成UI设计：

|颜色Key|日间模式|深色模式|备注|
|-|-|- ||
|skinPrimaryTextColor|#000000|#FFFFFF|标题字体颜色|
|skinSecondaryTextColor|#CCCCCC|#CCCCCC|次级标题字体颜色|
|skinMainBgColor|#FFFFFF|#333333|页面主背景色|
|skinSecondaryBgColor|#EEEEEE|#000000|次级背景、分隔线背景色|
|其他更多...|| ||
|skinProgressBarColor|#000000|#FFFFFF|进度条颜色|

这对于开发人员的效率提升更加明显，开发者不再需要关心具体颜色的值，只需要将对应的`color`填充到布局中即可：

```xml
<TextView
    android:layout_width="wrap_content"
    android:layout_height="wrap_content"
    android:text="Hello World"
    android:textColor="@color/skinPrimaryTextColor" />
```

## 二、构建产品化思维：皮肤包

如何衡量一个开发人员的能力——对复杂功能快速、稳定的交付？

如果只是单纯的认可这个理念，那么对于换肤功能的实现反而简单了，以标题颜色`skinPrimaryTextColor`为例，我只需要声明两个`color`资源：

```xml
<?xml version="1.0" encoding="utf-8"?>
<resources>
    <color name="skinPrimaryTextColor">#000000</color>
    <color name="skinPrimaryTextColor_Dark">#FFFFFF</color>
</resources>
```

笔者成功摆脱了复杂的编码实现，在`Activity`中我只需2行代码即可：

```java
public void initView() {
  if (isLightMode) {    // 日间模式
     tv.setTextColor(R.color.skinPrimaryTextColor);
  } else {              // 夜间模式
     tv.setTextColor(R.color.skinPrimaryTextColor_Dark);
  }
}
```

这种实现并非一无是处，从实现的难度而言，至少能够保护开发者为数不多的发囊。

当然，这种方案有「优化空间」，比如提供封装的工具方法 **看似摆脱** 无尽的`if-else`：

```java
/**
 * 获取当前皮肤下真正的color资源，所有color的获取都必须通过该方法。
 **/
@ColorRes
public static int getColorRes(@ColorRes int colorRes) {
  // 伪代码
  if (isLightMode) {     // 日间模式
     return colorRes;    // skinPrimaryTextColor
  } else {               // 夜间模式
     return colorRes + "_Dark";   // skinPrimaryTextColor_Dark
  }
}

// 代码中使用该方法，设置标题和次级标题颜色
tv.setTextColor(SkinUtil.getColorRes(R.color.skinPrimaryTextColor));
tvSubTitle.setTextColor(SkinUtil.getColorRes(R.color.skinSecondaryTextColor));
```

> 很明显，`return colorRes + "_Dark"`这行代码作为`int`类型的返回值是不成立的，读者无需关注具体实现，因为这种封装仍 **未摆脱笨重的 if-else 实现** 的本质。

可以预见，随着主题数量逐步增多，换肤相关的代码越来越臃肿，最关键的问题是，所有控件的相关颜色都强耦合于换肤相关代码本身，每个UI容器（`Activity`/`Fragment`/自定义`View`）等需要追加`Java`代码手动设置。

此外，当皮肤数量达到一定规模时，`color`资源的庞大势必影响到`apk`体积，因此主题资源的动态加载发势在必行，用户安装应用时默认只有一个主题，其它主题 **按需下载和安装** ，比如淘宝：

![](taobao.png)

到了这里，**皮肤包** 的概念应运而出，开发者需要将单个主题的颜色资源视为一个 **皮肤包**，在不同的主题下，对不同的皮肤包进行加载和资源替换：

```xml
<!--日间模式皮肤包的colors.xml-->
<resources>
    <color name="skinPrimaryTextColor">#000000</color>
    ...
</resources>

<!--深色模式皮肤包的colors.xml-->
<resources>
    <color name="skinPrimaryTextColor">#FFFFFF</color>
    ...
</resources>
```

这样，对于业务代码而言，开发者不再需要关注具体是哪个主题，只需要按常规的方式进行颜色的指定，系统会根据当前的颜色资源对`View`进行填充：

```xml
<!--当前切换到什么主题，系统就用对应的颜色值进行填充-->
<TextView
    android:layout_width="wrap_content"
    android:layout_height="wrap_content"
    android:text="Hello World"
    android:textColor="@color/skinPrimaryTextColor" />
```

回到本小节最初的问题，产品化思维也是一个优秀的开发者不可或缺的能力：先根据需求罗列不同的实现方案，做出对应的权衡，最后动手编码。

## 三、整合思路

目前为止，一切都还停留在需求提出和设计阶段，随着需求的明确，技术难点逐一罗列在开发者面前。

### 1.动态刷新机制

开发者面临的第一个问题：如何实现换肤后的 **动态刷新** 功能。

以微信注册页面为例，手动切换到深色模式后，微信进行了页面的刷新：

![](hot_update_wechat.gif)

读者不禁会问，**动态刷新的意义是什么** ，让当前页面重建或者APP重启不行吗？

当然可行，但是 **不合理** ，因为页面重建意味着页面状态的丢失，用户无法接受一个表单页面已填信息被重置；而如果要弥补这个问题，对每个页面重建追加状态的保存（`Activity.onSaveInstanceState()`），在实现的角度来看，也是一个巨大的工程量。

因此动态刷新势在必行——用户无论是在应用内切换了皮肤包，还是手动切换了系统的深色模式，我们如何将这个通知进行下发，保证所有页面都完成对应的刷新呢？

### 2.保存所有页面的Activity

读者知道，我们可以通过`Application.registerActivityLifecycleCallbacks()`方法观察到应用内所有`Activity`的生命周期，这也意味着我们可以持有所有的`Activity`：

```java
public class MyApp extends Application {

    // 当前应用内的所有Activity
    private List<Activity> mPages = new ArrayList();

    @Override
    public void onCreate() {
        super.onCreate();
        registerActivityLifecycleCallbacks(new ActivityLifecycleCallbacks() {
            @Override
            public void onActivityCreated(@NonNull Activity activity, @Nullable Bundle savedInstanceState) {
              mPages.add(activity);
            }

            @Override
            public void onActivityDestroyed(@NonNull Activity activity) {
              mPages.remove(activity);
            }

            // ...省略其它生命周期
        });
    }
}
```

有了所有的`Activity`的引用，开发者就可以在接到换肤通知的时候，第一时间尝试让所有页面的所有`View`去更新换肤。

### 3.成本问题

但巨大的谜团随之映入眼帘，对于控件而言，**更新换肤这个概念本身并不存在**。

什么意思呢？ 当换肤通知到达时，我无法令`TextView`更新文字颜色，也无法令`View`更新背景颜色——它们都只是系统的控件，执行的都是最基础的逻辑，说白了，开发者根本无法进行编码。

有同学说，那我直接让整个页面的整个`View`树所有`View`都全部重新渲染可以吗？可以，但是又回到了最初的问题，那就是所有`View`本身的状态也被重置了（比如`EditText`的文字被清零），退一步讲，即使这一点可以被接受，那么整个`View`树的重新渲染也会极大影响性能。

那么，如何尽可能的 **节省页面动态刷新的成本** ？

开发者希望，换肤发生时，**只对指定控件的指定属性进行动态更新**，比如，`TextView`只关注更新`background`和`textColor`，`ViewGroup`只关注`background`，其他的属性不需要重置和修改，将设备的每一分性能都利用到极致：

```java
public interface SkinSupportable {
  void updateSkin();
}

class SkinCompatTextView extends TextView implements SkinSupportable {

  public void updateSkin() {
    // 使用当前最新的资源更新 background 和 textColor
  }
}

class SkinCompatFrameLayout extends FrameLayout implements SkinSupportable {

  public void updateSkin() {
    // 使用当前最新的资源更新 background
  }
}
```

如代码所示，`SkinSupportable`是一个接口，实现该接口的类意味着都支持动态刷新，当换肤发生时，我们只需要拿到当前的`Activity`，并通过遍历`View`树，让所有`SkinSupportable`的实现类都去执行`updateSkin`方法进行自身的刷新，那么整个页面也就完成了换肤的刷新，同时不会影响`View`本身当前其他的属性。

当然，这也意味着开发者需要将常规的控件进行一轮覆盖性的封装，并提供出对应的依赖：

```groovy
implementation 'skin.support:skin-support:1.0.0'                   // 基础控件支持，比如SkinCompatTextView、SkinCompatFrameLayout等
implementation 'skin.support:skin-support-cardview:1.0.0'          // 三方控件支持，比如SkinCompatCardView
implementation 'skin.support:skin-support-constraint-layout:1.0.0' // 三方控件支持，比如SkinCompatConstraintLayout
```

从长期来看，针对控件一一封装，提供可组合选择的依赖，对于换肤库的设计者而言，库本身的开发成本其实并不高。


### 4.牵一发而动全身

但负责业务开发的开发者叫苦不迭。

按照目前的设计，岂不是工程的`xml文件`中所有控件都需要重新进行替换？

```xml
<!--使用前-->
<TextView
    android:layout_width="wrap_content"
    android:layout_height="wrap_content"
    android:text="Hello World"
    android:textColor="@color/skinPrimaryTextColor" />

<!--需要替换为-->
<skin.support.SkinCompatTextView
    android:layout_width="wrap_content"
    android:layout_height="wrap_content"
    android:text="Hello World"
    android:textColor="@color/skinPrimaryTextColor" />
```

从另一个角度来看，这又是额外的成本，如果哪一天想要剔除或者替换换肤库，那么无异于一次新的重构。

因此设计者需要尽量避免类似 **牵一发而动全身** 的设计，最好是让开发者无感知的感受到换肤库的 **动态更新**。

### 5.着手点: LayoutInflater.Factory2

> 对 `LayoutInflater` 不了解的读者，可以参考笔者的 [这篇文章](https://github.com/qingmei2/blogs/issues/25) 。

了解`LayoutInflater`的读者应该知道，在解析`xml`文件并实例化`View`的过程中，`LayoutInflater`通过自身的`Factory2`接口，将基础控件拦截并创建成对应的`AppCompatXXXView`，既避免了反射创建`View`对性能的影响，也保证了向下的兼容性:

```java
switch (name) {
    // 解析xml，基础组件都通过new方式进行创建
    case "TextView":
        view = new AppCompatTextView(context, attrs);
        break;
    case "ImageView":
        view = new AppCompatImageView(context, attrs);
        break;
    case "Button":
        view = new AppCompatButton(context, attrs);
        break;
    case "EditText":
        view = new AppCompatEditText(context, attrs);
        break;
    // ...
    default:
    // 其他通过反射创建
}
```

一图以蔽之：

![](skin_layout_inflater.png)

因此，`LayoutInflater`本身的实现思路为我们提供了一个非常好的着手点，我们只需要对这段逻辑进行拦截，将控件的实例化委托给换肤库即可：

![](skin_skin_layout_inflater.png)

如图所示，我们使用`SkinCompatViewInflater`拦截替换了系统`LayoutInflater`本身的逻辑，以`CardView`为例，在解析到对应的标签时，将`CardView`生成的逻辑委托给下面的依赖库，如果工程中添加了对应的依赖，那么就能生成对应的`SkinCompatCardView`，自然也就支持了动态换肤功能。

当然，这一切逻辑的实现，起源于工程添加对应的依赖,然后在APP启动时进行初始化：

```groovy
implementation 'skin.support:skin-support:1.0.0'   
implementation 'skin.support:skin-support-cardview:1.0.0'
```

```java
// App.onCreate()
SkinCompatManager.withApplication(this)
                .addInflater(new SkinAppCompatViewInflater())   // 基础控件换肤
                .addInflater(new SkinCardViewInflater())        // cardView
                .init();     
```

这样，库的设计者为换肤库提供了足够的灵活性，既避免了对现有工程大刀阔斧的修改，又保证极低的使用和迁移成本，如果我希望 **移除** 或者 **替换** 换肤库，只需要删除`build.gradle`中的依赖和`Application`中初始化的代码就可以了。
