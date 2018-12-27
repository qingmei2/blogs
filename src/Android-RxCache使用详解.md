## 前言

### 我为什么使用这个库？
 事实上Android开发中缓存功能的实现选择有很多种，File缓存，SP缓存，或者数据库缓存，当然还有一些简单的库/工具类，比如github上的这个：
 
 [【ASimpleCache】:a simple cache for android and java](https://github.com/yangfuhai/ASimpleCache)
 
 但是都不是很好用(虽然可能学习成本比较低，因为它使用起来相对简单)，我可能需要很多的静态常量来作为key存储缓存数据value，并设置缓存的有效期，这可能需要很多Java代码去实现，并且过程繁琐。
 
 如果您使用的网络请求库是Retrofit+RxJava，那么我推荐使用RxCache,正如作者所说的：
 
>RxCache is a reactive caching library for Android and Java which turns your caching needs into an interface.
>
> RxCache是一个用于Android和Java的响应式缓存库，它可将您的缓存需求转换为一个接口。

### 为什么写这样一篇文章
因为这个库的官方文档是！英！语！的！
这本身无可厚非，作为一个开发者，英语文档的阅读是不可避免的一项技能，但是笔者还是抽了一点时间将官方文档做了汉化：

[RxCache官方文档中文翻译](http://blog.csdn.net/mq2553299/article/details/78056742)

[RxCache库官方链接](https://github.com/VictorAlbertos/RxCache)
 
 文档的翻译比想象中的费力（每一个词都试图翻译准确），但数小时的努力之后，译文的描述依然对于初次接触该库的开发者有着不小的学习难度，干脆自己写一个demo，并放到github上，供大家参考。
 
[【Github】本文demo源码,点击进入](https://github.com/qingmei2/RxFamilyUsage-Android)
 
 

## 1.依赖配置
在您的build.gradle（Project）中添加JitPack仓库：

```gradle
allprojects {
    repositories {
        jcenter()
        maven { url "https://jitpack.io" }
    }
}
```

将下列的依赖添加到Module的build.gradle中：

```gradle
dependencies {
    compile "com.github.VictorAlbertos.RxCache:runtime:1.8.1-2.x"
    compile "io.reactivex.rxjava2:rxjava:2.0.6"
	//我们再添加这个依赖，下面有说明
    compile 'com.github.VictorAlbertos.Jolyglot:gson:0.0.3'
}
```

因为RxCache在内部使用 [Jolyglot](https://github.com/VictorAlbertos/Jolyglot) 对对象进行序列化和反序列化, 您需要选择下列的依赖中选择一个进行添加：
 
```gradle
dependencies {
    // To use Gson 
    compile 'com.github.VictorAlbertos.Jolyglot:gson:0.0.3'
    
    // To use Jackson
    compile 'com.github.VictorAlbertos.Jolyglot:jackson:0.0.3'
    
    // To use Moshi
    compile 'com.github.VictorAlbertos.Jolyglot:moshi:0.0.3'
}
```

## 2.Retrofit请求示例
我们假设这样一个需求，通过传入user名，返回User对应信息，比如：
 

### Retrofit API接口

```java
public interface GitHubService {

    @GET("users/{user}")
    Observable<User> getRxUser(@Path("user") String user);

}
```

### 该API请求的管理类ServiceManager

```java
public class GitHubServiceManager {

    private GitHubService service;

    public GitHubServiceManager() {
        init();
    }

    private void init() {
        HttpLoggingInterceptor interceptor = new HttpLoggingInterceptor()
                .setLevel(HttpLoggingInterceptor.Level.BODY);

        OkHttpClient client = new OkHttpClient()
                .newBuilder()
                .addInterceptor(interceptor)
                .build();

        service = new Retrofit.Builder()
                .baseUrl("https://api.github.com/")
                .addConverterFactory(GsonConverterFactory.create())
                .addCallAdapterFactory(RxJava2CallAdapterFactory.create())
                .client(client)
                .build()
                .create(GitHubService.class);
    }

    public Observable<User> getUser(String user){
        return service.getRxUser(user);
    }
}
```

### User数据类

```
@Data	//lombok插件的注解，自动生成get、set方法
public class User {

  public String login;

  public String name;
}
```

### 最后在我们的Activity中获取数据：
```java
 new GitHubServiceManager()
                .getUser(userName)
                .subscribeOn(Schedulers.io())
                .observeOn(AndroidSchedulers.mainThread())
                .subscribe(user1 -> Toast.makeText(this, user1.toString(), Toast.LENGTH_SHORT).show());
```

ok,非常简单，接下来我们来配置缓存，我们默认需求：缓存有效期为1分钟。

## 3.缓存配置

## 配置缓存接口

 首先我们先配置Provider接口：
 
```java
public interface UserCacheProviders {

    /**
     * LifeCache设置缓存过期时间. 如果没有设置@LifeCache , 数据将被永久缓存理除非你使用了 EvictProvider,EvictDynamicKey or EvictDynamicKeyGroup .
     * @param user
     * @param userName 驱逐与一个特定的键使用EvictDynamicKey相关的数据。比如分页，排序或筛选要求
     * @param evictDynamicKey   可以明确地清理指定的数据 DynamicKey.
     * @return
     */
    @LifeCache(duration = 1,timeUnit = TimeUnit.MINUTES)
    Observable<User> getUser(Observable<User> user, DynamicKey userName, EvictDynamicKey evictDynamicKey);

}

```
很多同学到这里就有点蒙蒙的，不知道这些参数都是用来干嘛的，其实简单介绍一下就清楚了：

> * @param user：这是个Observable类型的对象，简单来说，这就是你将要缓存的数据对象。
> * @param userName:DynamicKey类型，顾名思义，就是一个动态的key，我们以它作为tag，将数据存储到对应名字的File中
> * @param evictDynamicKey   可以明确地清理指定的数据 ，很简单，如果我们该参数传入为true，那么RxCache就会驱逐对应的缓存数据直接进行网络的新一次请求（即使缓存没有过期）。如果传入为false，说明不驱逐缓存数据，如果缓存数据没有过期，那么就不请求网络，直接读取缓存数据返回。
> * @return 可以看到，该接口方法中，返回值为Observable,泛型为user，这个Observable的对象user和参数中传进来的Observable的对象user有什么区别呢？
--- 很简单，返回值Observable中的数据为经过缓存处理的数据。

### 配置缓存Provider
我们还需要配置的有：
1.缓存文件存储到哪里？
2.如何解析缓存数据？
```
public class CacheProviders {

    private static UserCacheProviders userCacheProviders;

    public synchronized static UserCacheProviders getUserCache() {
        if (userCacheProviders == null) {
            userCacheProviders = new RxCache.Builder()
                    .persistence(BaseApplication.getApplication().getExternalCacheDir(), new GsonSpeaker())//缓存文件的配置、数据的解析配置
                    .using(UserCacheProviders.class);//这些配置对应的缓存接口
        }
        return userCacheProviders;
    }
}
```
 
### 代码中设置缓存功能：

```java
 private void requestHttp(String userName) {
		 //网络请求数据
        Observable<User> user = new GitHubServiceManager()
                .getUser(userName);
        //缓存配置        
        CacheProviders.getUserCache()
                .getUser(user, new DynamicKey(userName), new EvictDynamicKey(false))//用户名作为动态key生成不同文件存储数据，默认不清除缓存数据
                .subscribeOn(Schedulers.io())
                .observeOn(AndroidSchedulers.mainThread())
                .subscribe(user1 -> Toast.makeText(this, user1.toString(), Toast.LENGTH_SHORT).show());
}
```
 配置好后，如果没有缓存或者缓存失效，则请求网络数据，缓存并展示数据。
 如果有缓存数据且缓存未失效，则不加载网络数据，直接展示本地缓存数据。
