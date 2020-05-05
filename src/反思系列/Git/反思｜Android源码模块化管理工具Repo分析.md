# 反思｜Android源码模块化管理工具Repo分析

> **「反思」** 系列是笔者对于 **学习归纳** 一种新的尝试，其起源与目录请参考 [这里](https://github.com/qingmei2/android-programming-profile/blob/master/src/%E5%8F%8D%E6%80%9D%E7%B3%BB%E5%88%97/%E5%8F%8D%E6%80%9D%7C%E7%B3%BB%E5%88%97%E7%9B%AE%E5%BD%95.md) 。

## 起源

随着`Android`项目 **模块化** 或 **插件化** 项目业务的愈发复杂，开发流程中通过版本控制工具（比如`Git`）管理项目的成本越来越高。

以大名鼎鼎的 **[Android源代码开源项目](https://android.googlesource.com/)** （`Android Open-Source Project`，下文简称 `ASOP`）为例，截止2020年初，`Android10`的源码项目，其模块化分割出的 [子项目](https://android.googlesource.com/platform/manifest/+/refs/tags/android-10.0.0_r33/default.xml) 已接近800个，而每一个子项目都是一个独立的`Git`仓库。

这意味着`Git`的使用成本究竟有多高？如果开发者希望针对`AOSP`的一个分支进行开发，就需要手动将每个子项目进行`checkout`操作，如果本地分支尚未创建，开发者便需要手动地在每一个子项目里面去创建分支。

如此高昂的使用成本显然需要一种更自动化的方式去处理。为此，`Google`的工程师基于`Git`进行了一系列的代码补充，推出了名为`Repo`的代码版本管理工具，其本质是通过`Python`开发出一系列的脚本命令，便于开发者对复杂的模块化源码项目进行统一的调度和切换。

即使对于上文说到的`AOSP`而言，其同样使用了`Repo`工具进行项目的管理，由此可见，对于 **高度模块化** 开发的`Android`项目而言，`Repo`工具的确有一定的学习和借鉴意义。

本文以`AOSP`为例，对`Repo`工具的 **使用流程** 和 **原理** 进行系统性的分析，读者需要对`Git`和`Repo`工具有一定的了解。

> 官方文档：Repo入门及基本使用
https://source.android.com/source/downloading.html

本文大纲如下：

![](https://github.com/qingmei2/qingmei2-blogs-art/blob/master/qingmei2-blogs-art/blogs/2020/image.sub2htlgte.png?raw=true)

## 核心思想

`Repo` 是以 `Git` 为基础构建的代码库管理工具。其并非用来取代 `Git`，只是为了让开发者在多模块的项目中更轻松地使用 `Git`。`Repo` 命令是一段可执行的 `Python` 脚本，开发者可以使用 `Repo` 执行跨网络操作。例如，借助单个 `Repo` 命令，将文件从多个代码库下载到本地工作目录。

那么，`Repo`幕后原理究竟是怎么样的？想要真正的理解`Repo`，就必须理解`Repo`最核心的三个要素：**Repo仓库**、**Manifest仓库** 以及 **项目源码仓库**。

这里我们先将三者的关系通过一张图进行概括，该图已经将`Repo`工具本身的结构描述的淋漓尽致：

![](https://raw.githubusercontent.com/qingmei2/qingmei2-blogs-art/master/blogs/2019/image.u752zwemith.png)

### 1、项目源码仓库：底层的被执行者

对于若干个模块化的子项目，也就是 **项目源码仓库** 而言，它们是开发者希望的 **被统一调度的对象**。

> 比如，通过一个简单的`Repo`命令，统一完成所有子项目的分支切换、代码提交、代码远端更新等等。

因此，对于`Repo`工具整个框架的设计而言，**项目源码仓库** 明显应该处于最底层，它们是被`Repo`命令执行操作的最基本元素。

### 2、Manifest仓库：子项目元信息的容器

**Manifest仓库** 中最重要的是一个名为`manifest.xml`的清单文件，其存储了所有子项目仓库的元信息。

当`Repo`命令想要对所有子项目进行对应操作的时候，其总是需要知道 **要操作的项目的相关信息**——比如，我想要`clone AOSP`所有子项目的代码，首先我需要知道所有子项目仓库的名称和仓库地址；这时，`Repo`便会从`manifest`仓库中获取对应所有仓库的元信息，并进行对应的`fetch`操作。

> 对于`Android`应用的开发者而言这很好理解，对于一个`APP`而言，其对应的组件通过在`manifest`中声明进行管理。

因此，想要通过`Repo`对模块化项目进行管理，项目的管理者必须提供一个对应的`manifest`清单文件，里面存储所有子项目的相关信息，这样，`Repo`工具才能通过对其进行解析，然后完成子项目的统一管理。

此外，读者应该知道，`AOSP`也是在迭代过程中不断变化的，因此，其每一个分支版本所包含的子项目信息可能都是不同的，这意味着`Manifest`仓库同样也是一个`Git`仓库，以达到`AOSP`不同分支版本中，该仓库对应存储的子项目元信息不同的目的。

### 3、Repo仓库：顶层命令的容器

`Repo`工具实际上是由一系列的`Python`脚本组成的，这些`Python`脚本通过调用`Git`命令来完成自己的功能。

`Repo`仓库的本质就是存储了各种各样的`Python`脚本，当开发者调用相关的`Repo`命令时，便会从`Repo`仓库中运行对应的脚本进行处理，并根据脚本中的代码逻辑，找到`manifest`中所有项目的元信息，然后将其中包含的子项目进行对应命令的处理——因此，我们可以称 **Repo仓库是顶层命令的容器**。

此外，和`Manifest`仓库相同，组成`Repo`工具的`Python`脚本本身也是一个`Git`仓库；每当开发者执行`Repo`命令的时候，`Repo`仓库都会对自己进行一次更新。

> 读者请务必深刻理解这三者的意义，这也是`Repo`工具内部最核心的三个概念，也是阅读下文内容的基础。

现在，通过`Repo`工具完成项目模块化的管理需要分步构建以上三个角色，但是在这之前，我们需要先将`Repo`工具添加到自己的开发环境中。

## 一、Repo脚本初始化流程

正如 [官方文档](https://source.android.com/source/downloading.html) 所描述的，通过以下命令安装`Repo`工具，并确保它可执行：

```shell
curl https://storage.googleapis.com/git-repo-downloads/repo > ~/bin/repo
chmod a+x ~/bin/repo
```

安装成功后，对应的目录下便会存在一个`repo`脚本文件，通过将其配置到环境中，开发者可以在终端中使用`repo`的基本命令。

整个流程如下图所示：

![](https://raw.githubusercontent.com/qingmei2/qingmei2-blogs-art/master/qingmei2-blogs-art/blogs/2020/repo1.mawwpqcynj.png)

## 二、Repo仓库创建流程

`Repo`脚本初始化完毕，接下来针对`Repo`仓库创建流程进行简单的分析。

### 1、工欲善其事，必先利其器

以`AOSP`项目为例，开发者通过以下命令来安装一个Repo仓库：

```shell
repo init -u https://android.googlesource.com/platform/manifest -b master
```

这个命令实际上是包含了两个操作：初始化 **Repo仓库** 和 **Manifest仓库**，其中`Repo`仓库完成初始化之后，才会继续初始化`Manifest`仓库。

这很好理解，`Repo`仓库的本质就是存储了各种各样的`Python`脚本，若它没有初始化，就不存在所谓的`Repo`相关命令，更遑论后面的`Manifest`仓库初始化和子项目代码初始化的流程了。

这一小节我们先分析 **Repo仓库** 的安装过程，在下一小节再分析 **Manifest仓库** 的安装过程。

本小节整体流程如下图所示：

![](https://raw.githubusercontent.com/qingmei2/qingmei2-blogs-art/master/qingmei2-blogs-art/blogs/2020/repo2_init_repo.qcbmi5wg5pp.png)

### 2、init命令分析

上一节我们成功安装了`repo`脚本文件，这个脚本里面提供了例如`version`、`help`、`init`等最基本的命令：

```Python
def main(orig_args):
    if cmd == 'help':                   // help命令
      _Help(args)
    if opt.version or cmd == 'version': // version命令
      _Version()
    if not cmd:
      _NotInstalled()
    if cmd == 'init' or cmd == 'gitc-init':  // init命令
      ...
```

由此可见`Repo`脚本最初提供的命令确实非常少，前两个命令十分好理解，分别是查看`Repo`工具相关依赖的版本或者查看帮助，比较重要的是`init`命令，这个命令的作用便是对本地`Repo`仓库的初始化。

那么`Repo`仓库如何才能初始化呢？设计者并没有尝试直接向远端服务器请求拉取代码，而是从当前目录开始 **往上遍历直到根目录** ，若在这个过程中找到一个`.repo/repo`目录，并且该目录本身的确是一个`Repo`仓库，便尝试从该仓库 **克隆一个新的`Repo`仓库** 到执行`Repo`脚本的目录中。

反之，若从本地向上直到根目录不存在`Repo`仓库，则尝试向远端克隆一个新的`Repo`仓库到本地来。

回到本地克隆`Repo`仓库的流程中，代码是如何判断本地的`.repo/repo`目录的确是一个`Repo`仓库的呢，代码中已经描述的非常清晰了：

```Python
def _RunSelf(wrapper_path):
  my_dir = os.path.dirname(wrapper_path)
  my_main = os.path.join(my_dir, 'main.py')
  my_git = os.path.join(my_dir, '.git')

  if os.path.isfile(my_main) and os.path.isdir(my_git):
    for name in ['git_config.py',
                 'project.py',
                 'subcmds']:
      if not os.path.exists(os.path.join(my_dir, name)):
        return None, None
    return my_main, my_git
  return None, None
```

>   从这里我们就可以看出，判断的依据是对应的需要满足以下条件：
> 1、存在一个.git目录；
  2、存在一个main.py文件；
  3、存在一个git_config.py文件；
  4、存在一个project.py文件；
  5、存在一个subcmds目录。

读到这里，读者可以对`Repo`仓库进行一个简单的总结了。

### 3、Repo仓库到底是什么

从上文的源码中，读者了解了`Repo`脚本源码中判断是否是`Repo`仓库的五个依据，从这些判断条件中，我们可以简单对`Repo`仓库的定位进行一个总结。

首先，从条件1中我们得知，组成`Repo`工具的`Python`脚本本身也是一个`Git`仓库；每当开发者执行`Repo`命令的时候，`Repo`仓库都会对自己进行一次更新。

其次，`Repo`仓库本身作为存储`Python`脚本的容器，其内部必然存在一个入口的`main`函数可供运行。

对于条件3而言，我们直到`Repo`工具本质是对`Git`命令的封装，因此，必须有一个类负责`Git`相关的配置信息，和提供简单的`Git`相关工具方法，这便是`git_config.py`文件的作用。

对于条件4，`Repo`仓库目录下还需要一个`project.py`文件，负责`Hook`相关功能，细心的读者应该注意到，`/.repo/repo`目录下还有一个`/hooks/`目录。

最后也是最重要的，`/.repo/repo`目录下必须还存在一个`subcmds`目录，顾名思义，这个目录下存储了绝大多数`repo`重要的命令，比如`sync`、`checkout`、`pull`、`commit`等等；这也说明了，**如果没有`Repo`仓库的初始化，使用`Repo`命令操作子项目代码仓库便是无稽之谈**。

## 三、Manifest仓库创建流程

继续回到上一节我们使用到的命令：

```shell
repo init -u https://android.googlesource.com/platform/manifest -b master
```

读者已经知道，通过`init`命令，我们在指定的目录下，成功初始化了`Repo`仓库。当安装好`Repo`仓库之后，就会调用该`Repo`仓库下面的`main.py`脚本，对应的文件为`.repo/repo/main.py`。

这样我们便可以通过`init`后面的`-u -b`参数，进行`Manifest`仓库的创建流程，其中`-u`指的是`manifest`文件所在仓库对应的`Url`地址，`-b`指的是对应仓库的默认分支。

本小节整体流程如下图所示：

![](https://raw.githubusercontent.com/qingmei2/qingmei2-blogs-art/master/qingmei2-blogs-art/blogs/2020/image.dzuzaqidhyq.png)

### 1、定义manifest文件

上文中我们提到，想要通过`Repo`对模块化项目进行管理，项目的管理者必须提供一个对应的`manifest`清单文件，里面存储所有子项目的相关信息，这样，`Repo`工具才能通过对其进行解析，然后完成子项目的统一管理。

对于公司的业务而言，项目的管理者需要根据自己公司的实际业务模块构造出自己的`manifest`文件，并放置在某个`git`仓库内，这样开发者便可以通过指定对应的`Url`构建`Manifest`仓库。

本文以`AOSP`项目为例，其项目清单文件所在的`Url`为：

> https://android.googlesource.com/platform/manifest

### 2、初始化Manifest仓库

通过`init`命令和对应的参数，`Repo`便可以尝试从远端克隆`Manifest`仓库，然后从指定的`Url`克隆对应的`manifest`文件，切换到对应的分支并进行解析。

这里描述比较简单，实际上内部实现逻辑非常复杂；比如，在向远端克隆对应的`Manifest`仓库之前，会先进行本地是否存在`Manifest`仓库的判断，若已经存在，则尝试更新本地的`Manifest`仓库，而非直接向远程仓库中克隆。此外，当未指定分支时，则会`checkout`一个`default`分支。

这之后，`Repo`会根据远端的`xml`清单文件尝试构建自己本地的`Manifest`
仓库。

### 3、Manifest仓库的文件层级

让我们看以下`/.repo/`目录下文件层级：

![](https://raw.githubusercontent.com/qingmei2/qingmei2-blogs-art/master/qingmei2-blogs-art/blogs/2020/repo3_init_index.z5174ewbial.png)

上文我们说到，`Manifest`仓库本身也是一个`Git`仓库，因此，当我们打开`.repo/manifests/`目录时，里面会存在一个`.git`的文件夹，远端的`Manifest`文件仓库中的所有文件都被克隆到了这个目录下。

这里重点说一下项目的`Git`仓库目录和工作目录的概念。一般来说，一个项目的`Git`仓库目录（默认为`.git`目录）是位于工作目录下面的，但是`Git`支持将一个项目的`Git`仓库目录和工作目录分开来存放。

在`AOSP`中，`Repo`仓库的`Git`目录位于工作目录`.repo/repo`下，`Manifest`仓库的`Git`目录有两份拷贝，一份`.git`位于工作目录`.repo/manifests`下，另外一份位于`.repo/manifests.git`目录。

同时，我们看到这里还有一个`.repo/manifest.xml`文件，这个文件是最终被`Repo`的文件，它是通过将`.repo/manifest`文件夹下的文件和`local_manifest`文件进行合并后生成的，关于`local_manifest`机制我们后文会讲到，这里仅需将`.repo/manifest.xml`文件视为最终被使用的配置文件即可。

### 4、解析并生成Projects项目

回到上图，我们知道名字带有`manifest`相关的文件和文件夹代表了`Manifest`仓库，其内部存储了所有子项目仓库的元信息；而`repo`文件夹中存储了`repo`相关命令的脚本文件。

读者注意到，除此之外，还有一部分名字带有`project`的文件和文件夹，它们便是代表了`Repo`解析`Manifest`后生成的子项目信息和文件。

在`Repo`中，其管理的所有子项目，每一个子项目都被封装成为了一个`Project`对象，该对象内部存储了一系列相关的信息。

现在，`Manifest`仓库被创建并初始化完毕，接下来我们分析`Repo`的`sync`流程，看看子项目是如何被统一下载和管理的。

## 四、子项目仓库Sync流程

执行完成`repo init`命令之后，我们就可以继续执行`repo sync`命令来克隆或者同步子项目了：

```shell
repo sync
```

当执行`repo sync`命令时，会默认尝试拉取远程仓库下载更新本地的`Manifest`
仓库，下载远端对应的`default.xml`文件。

下载完成后，会自动解析`default.xml`文件中项目管理者配置的所有子项目信息，然后每个子项目信息被解析成为一个`Project`对象，并整合到一个内存的集合中去。

接下来，根据本地是否已经存在对应的子项目源码，针对每一个子项目，`Repo`都会进行对应的更新操作或者克隆操作，而这些操作的本质，其实就是内部调用了`Git`的`fetch`、`rebase`或者`merge`等等命令。

值得关注的是，和`Manifest`仓库相似，`AOSP`子项目的工作目录和`Git`目录也都是分开存放的，其中，工作目录位于`AOSP`根目录下，`Git`目录位于`.repo/projects`目录下。

此外，每一个`AOSP`子项目的工作目录也有一个`.git`目录，不过这个`.git`目录是一个符号链接，链接到`.repo/repo/projects`对应的`Git`目录。这样，我们就既可以在`AOSP`子项目的工作目录下执行`Git`命令，也可以在其对应的`Git`目录下执行`Git`命令。

本小节整体流程如下图所示：

![](https://raw.githubusercontent.com/qingmei2/qingmei2-blogs-art/master/qingmei2-blogs-art/blogs/2020/image.vyrg06ottoq.png)

## 五、LocalManifest机制

从上文中读者已经知道了，对于源码来讲，`manifest.xml`只是一个到`.repo/manifests/default.xml`的文件链接，真正的清单文件是通过`manifests`这个`Git`仓库托管起来的。

需要注意的是，在进行`Android`系统开发时，通常需要对清单文件进行自定义定制。例如，设备厂商会构建自己的`manifest`库，通常是基于`AOSP`的`default.xml`进行定制，去掉`AOSP`的一些`Git`库、增加一些自有的`Git`库。

这意味着，项目的管理者需要手动的对`default.xml`文件内容进行修改，然而这种方式在一些场景下存在弊端——对于`AOSP`而言，其本身可能存在几百个不同的分支，而项目的管理者需要修改的内容却基本是相同的。

> 比如，国内某个手机厂商需要删除`AOSP`中某个不受中国支持的功能，就需要对每个分支的`default.xml`文件内容进行相同的修改——删除某个`project`标签。

因此，`Repo`工具提出了另外一种本地的支持，这个机制便是`LocalManifest`机制。

在`repo sync`下载代码之前，会将`.repo/manifests/default.xml、local_manifest.xml`和`.repo/local_manifests/`目录下存在清单文件进行合并，再根据融合的清单文件进行代码同步。

这样一来，只需要将清单文件的修改项放到`.repo/local_manifests/`目录下， 就能够在不修改`default.xml`的前提下，完成对清单的文件的定制。

`LocalManifest`机制的原理图如下所示：

![](https://img-blog.csdn.net/20171121164108326?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvR2F1Z2FtZWxh/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

参考网上的资料，`Local Manifests`的隐含规则如下：
* 1、先解析`local_manifest.xml`，再解析`local_manifests/`目录下的清单文件;
* 2、`local_manifests`目录下的清单文件是没有命名限制的，但会按照字母序被解析，即字母序靠后的文件内容会覆盖之前的;
* 3、 所有清单文件的内容必须遵循`repo`定义的格式才能被正确解析。

## 参考 & 感谢

> 1.《Android源代码仓库及其管理工具Repo分析》 by 罗升阳:  
https://blog.csdn.net/Luoshengyang/article/details/18195205

罗老师的这篇文章非常经典，文章针对源码进行了非常细致的讲解，本文前四个小节都是参考该文进行的参考总结，强烈建议阅读。

> 2.《Android Local Manifests机制》 by ZhangJianIsAStark:  
https://blog.csdn.net/gaugamela/article/details/78593000

针对 **LocalManifests机制** 进行了非常详细的讲解，本文的第五节内容都是从中截取的，想要仔细了解的可以阅读本文。

> 3.AOSP Google 官方文档：  
https://source.android.com/source/developing.html

> 4.《Google Git-Repo 多仓库项目管理》 by 郑晓鹏-Rocko:  
https://juejin.im/post/5bf5913fe51d457dd7800a73

一篇非常不错的实践总结，该文并非针对`Repo`进行系统性的讲述，但是对于实践者而言是一篇不错的参考文章，从基础到集成到`jenkins`都有讲述。

## 关于我

Hello，我是 [却把清梅嗅](https://github.com/qingmei2)，女儿奴，源码的眷者，观众途径序列1，杀人游戏信徒，大头菜投机者，端茶递水工程师。欢迎关注我的 [博客](https://juejin.im/user/588555ff1b69e600591e8462/posts) 或者 [GitHub](https://github.com/qingmei2)。

如果您觉得文章对您有价值，欢迎 ❤️，或通过下方打赏功能，督促我写出更好的文章 :)

* [我的学习体系](https://github.com/qingmei2/blogs)
* **[关于知识付费](https://github.com/qingmei2/blogs/blob/master/appreciation.md)**
* **[关于「反思」系列](https://github.com/qingmei2/blogs/blob/master/src/%E5%8F%8D%E6%80%9D%E7%B3%BB%E5%88%97/%E5%8F%8D%E6%80%9D%7C%E7%B3%BB%E5%88%97%E7%9B%AE%E5%BD%95.md)**
