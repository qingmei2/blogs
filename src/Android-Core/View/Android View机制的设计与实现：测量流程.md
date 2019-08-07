# Android | View机制设计与实现：测量流程

`Android`本身的`View`体系非常庞大，源码中值得思考和借鉴之处众多，以`View`本身的绘制流程为例，其经过`measure`测量、`layout`布局、`draw`绘制三个过程，最终才能够讲一个`View`绘制出来并展示在用户面前。

本文将针对绘制过程中的 **测量流程(`Measure`)** 进行系统地归纳总结：

![](https://raw.githubusercontent.com/qingmei2/qingmei2-blogs-art/master/android/core/view/image.mcipzl9ajip.png)

## 概述

`View`的测量机制本质非常简单，顾名思义，其目的便是 **测量控件的宽高值**，围绕该目的，`View`的设计者通过代码编织了一整套复杂的逻辑：

* 1、对于子`View`而言，其本身宽高直接受限于父`View`的 **布局要求**，举例来说，父`View`被限制宽度为`40px`,子`View`的最大宽度同样也需受限于这个数值。因此，在测量子`View`之时，子`View`必须已知父`View`的布局要求，这个 **布局要求**，  `Android`中通过使用 `MeasureSpec` 类来进行描述。

* 2、对于父控件的测量流程而言，其必然依赖所有子控件宽高测量——即`measure`步骤的完成；若子控件本身未测量完毕，父控件自身的测量自然也无从谈起。因此，`Android`中`View`的测量流程中使用了非常经典的 **递归思想**：对于一个完整的界面而言，每个页面都映射了一个`View`树，其最顶端的`ViewGroup`测量开始时，会通过 **遍历** 将其 **布局要求** 传递给子`ViewGroup`进行子`ViewGroup`的测量，子`ViewGroup`在测量过程中也会通过 **遍历** 将其 **布局要求** 传递给其所有的子`View`...这种通过**遍历**自顶向下传递数据的方式我们称为 **测量过程中的“递”流程**。而当最底端树叶位置的子`View`自身测量完毕后，其父控件会将所有子控件的宽高数据进行聚合，然后通过对应的 **测量策略** 计算出父控件本身的宽高，测量完毕后，父控件的父控件也会根据其所有子控件的测量结果对自身进行测量，这种从底部向上传递各自的测量结果，最终完成`View`树最顶端`ViewGroup`的测量方式我们称为**测量过程中的“归”流程**，至此界面整个`View`树测量完毕。

对于绘制流程不甚熟悉的开发者而言，上述文字似乎晦涩难懂，但这些文字的概括其本质却是绘制流程整体的设计思想，**读者不应该将本文视为源码分析，而应该将自己代入到设计的过程中** ，当深刻理解整个流程的设计思路之后，测量流程代码地设计和编写自然行云流水一气呵成。

## 布局要求

在整个 **测量流程** 中， **布局要求** 都是一个非常重要的核心名词，`Android`中通过使用 `MeasureSpec` 类来对其进行描述。

为什么说 **布局要求** 非常重要呢，其又是如何定义的呢？这要先从结果说起，对于单个`View`来说，测量流程的结果无非是获取控件自身宽和高的值，`Android`提供了`setMeasureDimension()`函数，开发者仅需要将测量结果作为参数并调用该函数，便可以视为`View`完成了自身的测量：

```Java
protected final void setMeasuredDimension(int measuredWidth, int measuredHeight) {
 // measuredWidth 测量结果，View的宽度
 // measuredHeight 测量结果，View的高度
 // 省略其它代码...

 // 该方法的本质就是将测量结果存起来，以便后续的layout和draw流程中获取控件的宽高
 mMeasuredWidth = measuredWidth;
 mMeasuredHeight = measuredHeight;
}
```

需要注意的是，子控件的测量过程本身还应该依赖于父控件的一些布局约束，比如：

* 1.父控件固定宽高只有`${x}px`，子控件设置为`layout_height="${y}px"`;  
* 2.父控件高度为`wrap_content`(包裹内容)，子控件设置为`layout_height="match_parent"`;  
* 3.父控件高度为`match_parent`(包裹内容)，子控件设置为`layout_height="match_parent"`;

这些情况下，简单的通过`setMeasuredDimension()`函数似乎都不可能达到测量控件的目的，因为无法计算出准确控件本身的宽高值, 即**子控件的测量须依赖于父控件和子控件两者共同才能完成**，而父控件对子控件的布局约束，便是前文提到的 **布局要求**，即`MeasureSpec`类。

## MeasureSpec类

从面向对象的角度来看，我们将`MeasureSpec`类设计成这样：

```Java
public final class MeasureSpec {

  int size;     // 测量大小
  Mode mode;    // 测量模式

  enum Mode { UNSPECIFIED, EXACTLY, AT_MOST }

  MeasureSpec(Mode mode, int size){
    this.mode = Mode;
    this.size = size;
  }

  public int getSize() { return size; }

  public Mode getMode() { return mode; }
}
```

在设计的过程中，我们将布局要求分成了2个属性。**测量大小** 意味着控件需要对应大小的宽高，**测量模式** 则表示控件对应的宽高模式：

> * UNSPECIFIED：父元素不对子元素施加任何束缚，子元素可以得到任意想要的大小；日常开发中自定义View不考虑这种模式，可暂时先忽略；
> * EXACTLY：父元素决定子元素的确切大小，子元素将被限定在给定的边界里而忽略它本身大小；这里我们理解为控件的宽或者高被设置为 `match_parent` 或者指定大小，比如`20dp`.
> * AT_MOST：子元素至多达到指定大小的值；这里我们理解为控件的宽或者高被设置为`wrap_content`。

巧妙的是，`Android`并非通过上述定义`MeasureSpec`对象的方式对 **布局要求** 进行描述，而是使用了更简单的二进制的方式，用一个32位的`int`值进行替代：

```Java
public static class MeasureSpec {
    private static final int MODE_SHIFT = 30; //移位位数为30
    //int类型占32位，向右移位30位，该属性表示掩码值，用来与size和mode进行"&"运算，获取对应值。
    private static final int MODE_MASK  = 0x3 << MODE_SHIFT;

    //00左移30位，其值为00 + (30位0)
    public static final int UNSPECIFIED = 0 << MODE_SHIFT;
    //01左移30位，其值为01 + (30位0)
    public static final int EXACTLY     = 1 << MODE_SHIFT;
    //10左移30位，其值为10 + (30位0)
    public static final int AT_MOST     = 2 << MODE_SHIFT;

    // 根据size和mode，创建一个测量要求
    public static int makeMeasureSpec(int size, int mode) {
        return size + mode;
    }

    // 根据规格提取出mode，
    public static int getMode(int measureSpec) {
        return (measureSpec & MODE_MASK);
    }

    // 根据规格提取出size
    public static int getSize(int measureSpec) {
        return (measureSpec & ~MODE_MASK);
    }
}
```

这个`int`值中，前2位代表了测量模式，后30位则表示了测量的大小，对于模式和大小值的获取，只需要通过位运算即可。

以宽度举例来说，若我们设置宽度=5px（二进制的101），那么`mode`对应`EXACTLY`,在创建测量要求的时候，只需要通过二进制的相加，便可得到存储了相关信息的`int`值：

![](https://raw.githubusercontent.com/qingmei2/qingmei2-blogs-art/master/android/core/view/image.29lx5bauvgy.png)

而当需要获得`Mode`的时候只需要用`measureSpec`与`MODE_TASK`相与即可如下图：

![](https://raw.githubusercontent.com/qingmei2/qingmei2-blogs-art/master/android/core/view/image.9gd1lev69d.png)

同理，想获得`size`的话只需要只需要`measureSpec`与`~MODE_TASK`相与即可如下图

![](https://raw.githubusercontent.com/qingmei2/qingmei2-blogs-art/master/android/core/view/image.ppkjtxeikvp.png)
