最近将公司的项目部署了`Jenkins`持续集成，遇到了几个麻烦的点，其中之一就是对`JDK`和`Tomcat`的配置，特此记录。

> 本地系统：MacOS
远程系统：CentOS_7_04_64_20G_alibase_201701015.vhd

### 1.安装jdk

* `yum search java|grep jdk` 查看yum库中都有哪些jdk版本
* `yum install java-1.8.0-openjdk` 安装Java8

### 2.安装tomcat

首先将tomcat的安装包 [下载](https://tomcat.apache.org/download-80.cgi#8.5.34) 在对应文件夹下：

![](https://upload-images.jianshu.io/upload_images/7293029-1fb45774f7eb75b5.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

解压压缩包：

* `tar -zxvf apache-tomcat-8.5.34.tar.gz`

配置端口号，进入 `tomcat` 的 `conf` 目录下，修改 `server.xml` 文件:

![](https://upload-images.jianshu.io/upload_images/7293029-88a463028a127113.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

修改端口，默认 8080:

![](https://upload-images.jianshu.io/upload_images/7293029-f81b366c28f02153.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

进入`tomcat`的`bin`目录下，启动tomcat：

* `./startup.sh`

![](https://upload-images.jianshu.io/upload_images/7293029-474e5c599d99913a.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

因为笔者用的是阿里云，直接输入`ip:端口`并没有成功访问tomcat，需要配置对应的安全组策略：

> 本实例安全组 -> 配置规则 -> 添加安全组规则

![](https://upload-images.jianshu.io/upload_images/7293029-9135e120193681c1.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


这样再次通过`ip:端口`就能够成功访问。

