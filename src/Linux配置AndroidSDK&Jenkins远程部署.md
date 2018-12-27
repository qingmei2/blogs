最近将公司的项目部署了`Jenkins`持续集成，遇到了几个麻烦的点，其中之一就是将`Android SDK`进行配置在远程服务器（总结下来还是自己对Linux命令还不够熟悉），特此记录。

* 系统：	**Ubuntu Server 16.04.1 LTS 64位**
* 前置：完成`JDK`的环境搭建

## 1.下载SDK

[点击进入下载网址](http://tools.android-studio.org/index.php/sdk) 下载对应的 `android-sdk_r24.4.1-linux.tgz` 文件。

## 2.解压下载的压缩包

* `tar -zxvf android-sdk_r24.4.1-linux.tgz`

## 3.安装32位库
Android SDK中的adb程序是32位的，Ubuntu x64系统需要安装32位库文件，用于兼容32位的程序:

* `sudo apt-get install -y libc6-i386 lib32stdc++6 lib32gcc1 lib32ncurses5 lib32z1`

![](https://upload-images.jianshu.io/upload_images/7293029-856114b298ba01f1.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

## 4.配置环境变量

* `export ANDROID_SDK_HOME=/home/XXX/android/sdk/android-sdk-linux`
* `export PATH=$PATH:${ANDROID_SDK_HOME}/tools`
* `export PATH=$PATH:${ANDROID_SDK_HOME}/platform-tools`

![](https://upload-images.jianshu.io/upload_images/7293029-2fdd06ccb80abf29.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

通过 `vim /etc/profile` **查看** 或 **编辑** 环境变量的配置（或者直接通过`export`命令查看）：

![](https://upload-images.jianshu.io/upload_images/7293029-11a6f7a55b31677e.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

## 5.下载最新SDK工具

进入`tools`目录下，输入`./android -v list sdk`命令查看可下载更新的`SDK`列表:

![](https://upload-images.jianshu.io/upload_images/7293029-02069b6bf36a313f.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

![](https://upload-images.jianshu.io/upload_images/7293029-5396b51a0616338c.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

官方提供了一些参数供开发者选择性更新：

> Action "update sdk":
  Updates the SDK by suggesting new platforms to install if available.
Options:
  -f --force    Forces replacement of a package or its parts, even if something has been modified
  -u --no-ui    Updates from command-line (does not display the GUI)
  -o --obsolete Installs obsolete packages
  -t --filter   A filter that limits the update to the specified types of packages in the form of a comma-separated list of [platform, tool, platform-tool, doc, sample, extra]
  -s --no-https Uses HTTP instead of HTTPS (the default) for downloads
  -n --dry-mode Simulates the update but does not download or install anything

上述参数通过`android update sdk --filter <component> --no-ui`命令进行 **组件** 的过滤性筛选。

笔者选择了简单粗暴，直接通过`android update sdk --no-ui`命令下载所有版本的sdk。

## 6.将sdk配置到Jenkins

打开`Jenkins` 的 **系统配置**界面，将对应的SDK根目录配置给环境变量：

![](https://upload-images.jianshu.io/upload_images/7293029-8bda3e825e605d09.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

## 7.构建错误处理

### 缺少License

错误日志：

>  What went wrong:
A problem occurred configuring project ':xxx'.
> Failed to install the following Android SDK packages as some licences have not been accepted.
     build-tools;27.0.3 Android SDK Build-Tools 27.0.3
  To build this project, accept the SDK license agreements and install the missing components using the Android Studio SDK Manager.

解决方案：

将本地sdk目录下的`licenses`文件夹中的License文件传到远程服务器中：

![](https://upload-images.jianshu.io/upload_images/7293029-e94ab6f1e1f1cb93.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

### 对应版本的SDK Build-Tools不存在

> 错误日志：Failed to install the following SDK components:
      build-tools;27.0.3 Android SDK Build-Tools 27.0.3
  The SDK directory is not writable (/home/sdk/android-sdk-linux)


![](https://upload-images.jianshu.io/upload_images/7293029-ba9bae93fc5fe09e.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

解决方案，更新对应的BuildTools版本:

查看所有版本列表：

* `./android list sdk -a` 

![](https://upload-images.jianshu.io/upload_images/7293029-9b8939739a30b584.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

更新对应的27.0.3版本：

* `android  update sdk -u -t 7 -a` 

![](https://upload-images.jianshu.io/upload_images/7293029-7b0d79c33a0113a6.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
