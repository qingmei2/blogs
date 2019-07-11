学习`Flutter`一小段时间，对纯`Flutter`项目有了一些基本的了解，但更趋近实际开发的应该是将`Flutter`模块作为一个依赖库添加到原生的`Android`项目中。

**本文笔者将尝试分享个人针对`Flutter`与`Android`混编时的配置步骤，以及踩坑过程。**

## 一、初始化Flutter-Module

参考 [官方文档](https://github.com/flutter/flutter/wiki/Add-Flutter-to-existing-apps) ，首先需要确认`Flutter-Module`依赖库文件夹的位置，简单来说，这里有两种方式：

* 1.创建在项目的根目录下（**内部**）；
* 2.创建和项目文件夹的同一层级（**外部**），这也是官方推荐的方式。

其实这些方式没什么区别，但是个人更倾向于第二种，我们在项目文件夹的目录层级下对`Flutter-Module`文件夹进行 **创建** 并 **初始化**：

```shell
$ flutter create -t module module_flutter
```

成功后，`Flutter-Module`和`Android`项目本身应该是这样的（红框内的两个项目）：

![](https://raw.githubusercontent.com/qingmei2/qingmei2-blogs-art/master/android/flutter/%E6%B7%B7%E7%BC%96%E9%85%8D%E7%BD%AE/image.ayoc0j1z217.png)

## 二、配置Android项目

接下来我们需要将这个项目和刚刚创建的`module-flutter`进行依赖，我们先打开`Android`原生项目，并为项目根目录下的`settings.gradle`文件中添加如下配置：

```groovy
setBinding(new Binding([gradle: this]))
evaluate(new File(
        settingsDir.parentFile,
        'module_flutter/.android/include_flutter.groovy'
))
```

如果`module-flutter`模块是创建在项目内部，那么需要稍微改一改：

```groovy
setBinding(new Binding([gradle: this]))
evaluate(new File(
        settingsDir.path,
        'module_flutter/.android/include_flutter.groovy'
))
```

然后，我们需要打开`app`的`build.gradle`文件，添加对`flutter`的依赖：

```groovy
dependencies {
    implementation fileTree(dir: 'libs', include: ['*.jar'])
    implementation 'androidx.appcompat:appcompat:1.0.0'
    implementation 'androidx.annotation:annotation:1.0.0'

    ......

    implementation project(':flutter')
}
```

这样，对于简单的`Android`原生项目而言，`Flutter`已经配置成功了。

## 三、AndroidX的迁移

由于笔者的项目迁移了`AndroidX`, 但是低版本的`Flutter`命令生成的`module`默认依赖的是`support`包, 因此我们需要将默认`support`的依赖手动迁移到`AndroidX`。

> 截止笔者发文前，`Flutter`V1.7已经提供了对`AndroidX`的支持，当创建 `Flutter` 项目的时候，你可以通过添加 `--androidx` 来确保生成的项目文件支持`AndroidX`，详情参考[这里](https://juejin.im/post/5d26bf1f51882536124052c2)。

手动迁移的方式有两种：

* 1.通过`Android Studio` **自动迁移** 过去。

首先通过`Android Studio`打开`flutter-module`，这时候是不能直接迁移`AndroidX`的，需要通过`flutter` - `Open Android module in AS` 方式新打开一个窗口。

![](https://raw.githubusercontent.com/qingmei2/qingmei2-blogs-art/master/android/flutter/%E6%B7%B7%E7%BC%96%E9%85%8D%E7%BD%AE/image.z13k5env0wo.png)

这样编译成功后，就可以点击`Refactor` - `Migrate to AndroidX`进行迁移了，后续步骤网上有很多，不赘述。

![](https://raw.githubusercontent.com/qingmei2/qingmei2-blogs-art/master/android/flutter/%E6%B7%B7%E7%BC%96%E9%85%8D%E7%BD%AE/image.3dvu4urlzlm.png)

* 2.手动配置过去，这个方式也很简单，打开`Flutter`-`build.gradle`文件，对依赖进行更新：

```groovy
android {
    //...
    defaultConfig {
      // ...
      testInstrumentationRunner "androidx.test.runner.AndroidJUnitRunner"
   }
   // ...
 }

dependencies {
    testImplementation 'junit:junit:4.12'
    implementation 'androidx.appcompat:appcompat:1.0.0'
    implementation 'androidx.annotation:annotation:1.0.0'

    // ...
}
```

手动配置网上有很多博客，不赘述。

需要注意的是，一定要保证`Flutter`模块中对`AndroidX`相关依赖的版本和实际原生项目中相关依赖的版本是一致的，否则可能会导致依赖冲突。

## 四、多模块项目的配置

上文说到，简单的项目已经配置完毕了，但是多模块的项目来说则稍显复杂，比如笔者的项目：

![](https://raw.githubusercontent.com/qingmei2/qingmei2-blogs-art/master/android/flutter/%E6%B7%B7%E7%BC%96%E9%85%8D%E7%BD%AE/image.0qphzgtzjnr.png)

首先，需要在底层`library`(本文中是`library-core`)的`build.gradle`文件中添加对`flutter`的依赖：

```groovy
dependencies {
    // ...
    api project(':flutter')
}
```

添加之后并进行同步，原生的项目就会对`settings.gradle`文件中指向的`module-flutter`文件夹进行依赖。

同步、编译成功后，我运行了项目，但我很快遇到了问题：

```
[ERROR:flutter/runtime/dart_vm_data.cc(19)] VM snapshot invalid and could not be inferred from settings.
[ERROR:flutter/runtime/dart_vm.cc(241)] Could not setup VM data to bootstrap the VM from.
[ERROR:flutter/runtime/dart_vm_lifecycle.cc(89)] Could not create Dart VM instance.
[FATAL:flutter/shell/common/shell.cc(218)] Check failed: vm. Must be able to initialize the VM.
```

## 五、ProductFlavors的坑

这个问题纠结了很久，最后在 [这个issue中](https://github.com/flutter/flutter/issues/19818) 找到了答案:

![](https://raw.githubusercontent.com/qingmei2/qingmei2-blogs-art/master/android/flutter/%E6%B7%B7%E7%BC%96%E9%85%8D%E7%BD%AE/image.zz5r3y4r7yn.png)

经 **@yk3372** 大佬提示，原来是以为项目中配置了`ProductFlavors`, 因此，`Flutter`模块中对应的`build.gradle`文件也需要进行对应的配置，比如这样：

```groovy
buildTypes {
    release {}
    debug {}
}
flavorDimensions "environment"
productFlavors {
    dev {}
    qa {}
    prod {}
}
```

配置好之后，还需要手动将相关`module`的`ProductFlavors`配置相同，否则会提示一堆错误，比如我的一个原生的`module`依赖了`flutter`的`module`，它们就必须都保持同一个状态：

![](https://raw.githubusercontent.com/qingmei2/qingmei2-blogs-art/master/android/flutter/%E6%B7%B7%E7%BC%96%E9%85%8D%E7%BD%AE/image.z3au8l1ezfm.png)

？？？这是不是意味着所有的`module`的`build.gradle`都配置相同的`productFlavors`信息？

实践给予我答案，是的。

![](https://raw.githubusercontent.com/qingmei2/qingmei2-blogs-art/master/android/flutter/%E6%B7%B7%E7%BC%96%E9%85%8D%E7%BD%AE/image.mk9phnvrrwe.png)

虽然折腾了很久，还好前人栽树，后人乘凉，解决了问题还是`happy ending`, `Github`大法好。

## 六、更多Flutter混编姿势

本文提供了 **官方文档** 提供混合开发的集成方式，实际上，国内很多大厂都分享过相关的技术文章，这里也一并放出来：

* [闲鱼Flutter混合工程持续集成的最佳实践](https://yq.aliyun.com/articles/618599?spm=a2c4e.11153959.0.0.4f29616b9f6OWs)

* [头条Flutter混合工程实践](https://mp.weixin.qq.com/s/wdbVVzZJFseX2GmEbuAdfA)

* [2019 最前沿的几个 Flutter 实践：微信、咸鱼、美团](https://mp.weixin.qq.com/s/TyjwBASNvxnQNXtC3zCG1w)

---

## 关于我

Hello，我是[却把清梅嗅](https://github.com/qingmei2)，如果您觉得文章对您有价值，欢迎 ❤️，也欢迎关注我的[博客](https://www.jianshu.com/u/df76f81fe3ff)或者[Github](https://github.com/qingmei2)。

如果您觉得文章还差了那么点东西，也请通过**关注**督促我写出更好的文章——万一哪天我进步了呢？

* [我的Android学习体系](https://github.com/qingmei2/android-programming-profile)
* [关于文章纠错](https://github.com/qingmei2/Programming-life/blob/master/error_collection.md)
* [关于知识付费](https://github.com/qingmei2/Programming-life/blob/master/appreciation.md)
