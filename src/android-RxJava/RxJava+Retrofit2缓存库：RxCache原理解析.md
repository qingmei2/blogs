## 动机

RxCache相对于其它Android缓存库相比，入门难度较大，原因之一就是因为语言的不同，加上原文档本身内容就晦涩难懂，笔者也尝试直接翻译，但对于刚接触该库的开发者来说仍然有不低的门槛。

感谢[@Jessyan](http://www.jianshu.com/u/1d0c0bc634db),笔者当初在入门这个库的时候，有幸看到了[《你不知道的Retrofit缓存库RxCache》](http://www.jianshu.com/p/b58ef6b0624b)这篇文章，获益匪浅，在之后的项目中又尝试加入RxCache，近日又翻了翻源码，对RxCache略有所得，尝试在源码级上将RxCache这个库进行一次深入的解剖。


## 架构
 
 首先来一张RxCache的架构图，[图片来源](http://www.jianshu.com/p/b58ef6b0624b) 已获得作者[@Jessyan](http://www.jianshu.com/u/1d0c0bc634db)授权，再次感谢：

![](http://upload-images.jianshu.io/upload_images/2974769-6ec90c04f0c95ee0.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
 
>  RxCache使用注解来为Retrofit配置缓存信息,内部使用动态代理和Dagger来实现。

上述图片及文字基本已经涵盖RxCache的原理。

## 原理分析

###一.数据的保存

数据的保存包括内存层（Memory）和持久层（Persistence），RxCache通过这两个接口及其实现类进行数据的保存：

#### 1.持久层

持久层（Persistence）主要依赖Persistence接口的子类Disk（磁盘）进行数据的操作，这个类负责的包括：

* 数据的序列化和反序列化
* 缓存数据的加密
* 缓存数据的存储文件夹路径配置

等等，笔者对于源码分析的目标是，先看明白架构和设计思想，第一次不要过深执着于细节的实现。因此本文仅仅列出Persistence接口的抽象方法，并对其进行简单介绍，详细实现有兴趣的朋友可以去查看Disk.class进行源码查看。


```java
public interface Persistence {
	//保存对象数据
  void save(String key, Object object, boolean isEncrypted, String encryptKey);
	//保存被封装的数据对象Record<T>
  void saveRecord(String key, Record record, boolean isEncrypted, String encryptKey);
	//根据key将对应的持久层数据驱逐
  void evict(String key);
    //驱逐该API的所有持久层数据
  void evictAll();
	//获得所有的key
  List<String> allKeys();
	//以MB计算累计的缓存数据大小
  int storedMB();
	//根据key取出数据（就是取出缓存）
  <T> T retrieve(String key, Class<T> clazz, boolean isEncrypted, String encryptKey);
   //根据key取出封装了数据的Record<T>对象
  <T> Record<T> retrieveRecord(String key, boolean isEncrypted, String encryptKey);
}
```
其中，Record对象是封装缓存数据以及该缓存数据的一些配置信息,我们简单看一下其中部分源码：

```
public final class Record<T> {
  private Source source;
  private final T data;//data就是缓存数据Object
  private final long timeAtWhichWasPersisted;
  private final String dataClassName, dataCollectionClassName, dataKeyMapClassName;
  private Boolean expirable;

  //LifeTime requires to be stored to be evicted by EvictExpiredRecordsTask when no life time is available without a config provider
  private Long lifeTime;

  //Required by EvictExpirableRecordsPersistence task
  private transient float sizeOnMb;
}
```
可以看到，Record对象内部除了封装了缓存数据，此外还可以从中取得一些可能会用到的配置信息。

#### 2.内存层：

内存层（Memory）主要依赖Memory接口的实现类ReferenceMapMemory, 我们依然以Memory接口为例。
```
public interface Memory {
	//根据key获取对应Record数据
  <T> Record<T> getIfPresent(String key);
	//根据key保存数据到内存
  <T> void put(String key, Record<T> record);
	//获得key集合
  Set<String> keySet();
	//根据key驱逐内存中特定数据
  void evict(String key);
	//驱逐内存中所有数据
  void evictAll();
}
``` 
值得一提的是：
> RxCache的内存缓存使用的是Map,它就用这个标识符作为Key,put和get数据(本地缓存则是将这个标识符作为文件名,使用流写入或读取这个文件,来储存或获取缓存),如果储存和获取的标识符不一致那就取不到想取的缓存。

这一点我们可以从实现类ReferenceMapMemory.class中看出:

```
public final class ReferenceMapMemory implements Memory {
	//内存缓存本质上使用的是Map进行数据的操作
  private final Map<String, io.rx_cache2.internal.Record> referenceMap;

  public ReferenceMapMemory() {
    referenceMap = Collections.synchronizedMap(new io.rx_cache2.internal.cache.memory.apache.ReferenceMap<String, Record>());
  }
  //......省略相关代码
}
```
好的，现在我们已经初步将内存层和持久层的类和接口进行了基本的了解，接下来我们来考虑一下操作缓存数据的类TwoLayersCache.

### 二.数据的操作

#### 1.TwoLayersCache
回到最上方的图中，我们可以看到，TwoLayersCache这个类下方有很多的附属类，实际上，这个类的作用如其名，就是负责管理内存层和持久层的缓存数据：

```
@Singleton
public final class TwoLayersCache {
  private final EvictRecord evictRecord;
  private final io.rx_cache2.internal.cache.RetrieveRecord retrieveRecord;
  private final SaveRecord saveRecord;

  @Inject public TwoLayersCache(EvictRecord evictRecord, io.rx_cache2.internal.cache.RetrieveRecord retrieveRecord,
      SaveRecord saveRecord) {
    this.evictRecord = evictRecord;
    this.retrieveRecord = retrieveRecord;
    this.saveRecord = saveRecord;
  }
  //下面就是处理缓存的方法：
  //retrieveRecord 取出对应缓存
  public <T> Record<T> retrieve(String providerKey, String dynamicKey, String dynamicKeyGroup,
      boolean useExpiredDataIfLoaderNotAvailable, Long lifeTime, boolean isEncrypted) {
    return retrieveRecord.retrieveRecord(providerKey, dynamicKey, dynamicKeyGroup,
        useExpiredDataIfLoaderNotAvailable, lifeTime, isEncrypted);
  }
  //saveRecord 保存缓存
    public void save(String providerKey, String dynamicKey, String dynamicKeyGroup, Object data,
      Long lifeTime, boolean isExpirable, boolean isEncrypted) {
    saveRecord.save(providerKey, dynamicKey, dynamicKeyGroup, data, lifeTime, isExpirable,
        isEncrypted);
  }
	//evictRecord驱逐特定API的缓存
  public void evictProviderKey(final String providerKey) {
    evictRecord.evictRecordsMatchingProviderKey(providerKey);
  }  
    //evictRecord驱逐特定API下某个key对应的缓存
  public void evictDynamicKey(String providerKey, String dynamicKey) {
    evictRecord.evictRecordsMatchingDynamicKey(providerKey, dynamicKey);
  }
 //evictRecord驱逐特定API下某个keygroup对应的缓存
  public void evictDynamicKeyGroup(String key, String dynamicKey, String dynamicKeyGroup) {
    evictRecord.evictRecordMatchingDynamicKeyGroup(key, dynamicKey, dynamicKeyGroup);
  }
	//驱逐所有缓存
  public void evictAll() {
    evictRecord.evictAll();
  }
 }
```
我们来看一下TwoLayersCache的三个成员：

* evictRecord 驱逐缓存数据
* retrieveRecord 获取缓存数据
* saveRecord 保存缓存数据

这些成员看起来分工明确，确实如此，但是这些对象是怎么对内存层和持久层的数据进行操作的呢？

#### 2. Action
事实上，这三个对象都是Action的子类，Action是一个抽象类：

```
abstract class Action {
  private static final String PREFIX_DYNAMIC_KEY = "$d$d$d$";
  private static final String PREFIX_DYNAMIC_KEY_GROUP = "$g$g$g$";

  protected final Memory memory;//操作内存数据
  protected final Persistence persistence;//操作本地数据
	
	//每次Action的实例化都伴随着内存层Memory和持久层Persistence的两个对象的实例化
  public Action(Memory memory, Persistence persistence) {
    this.memory = memory;
    this.persistence = persistence;
  }
	
	//缓存存储时的命名规则
  protected String composeKey(String providerKey, String dynamicKey, String dynamicKeyGroup) {
    return providerKey
        + PREFIX_DYNAMIC_KEY
        + dynamicKey
        + PREFIX_DYNAMIC_KEY_GROUP
        + dynamicKeyGroup;
  }
	//根据对应的命名规则，取出对应缓存文件中对应的缓存数据
  protected List<String> getKeysOnMemoryMatchingProviderKey(String providerKey) {
    List<String> keysMatchingProviderKey = new ArrayList<>();

    for (String composedKeyMemory : memory.keySet()) {
      final String keyPartProviderMemory =
          composedKeyMemory.substring(0, composedKeyMemory.lastIndexOf(PREFIX_DYNAMIC_KEY));

      if (providerKey.equals(keyPartProviderMemory)) {
        keysMatchingProviderKey.add(composedKeyMemory);
      }
    }

    return keysMatchingProviderKey;
  }

  protected List<String> getKeysOnMemoryMatchingDynamicKey(String providerKey, String dynamicKey) {
    List<String> keysMatchingDynamicKey = new ArrayList<>();

    String composedProviderKeyAndDynamicKey = providerKey + PREFIX_DYNAMIC_KEY + dynamicKey;

    for (String composedKeyMemory : memory.keySet()) {
      final String keyPartProviderAndDynamicKeyMemory =
          composedKeyMemory.substring(0, composedKeyMemory.lastIndexOf(PREFIX_DYNAMIC_KEY_GROUP));

      if (composedProviderKeyAndDynamicKey.equals(keyPartProviderAndDynamicKeyMemory)) {
        keysMatchingDynamicKey.add(composedKeyMemory);
      }
    }

    return keysMatchingDynamicKey;
  }
	//获得缓存规则的字符串
  protected String getKeyOnMemoryMatchingDynamicKeyGroup(String providerKey, String dynamicKey,
      String dynamicKeyGroup) {
    return composeKey(providerKey, dynamicKey, dynamicKeyGroup);
  }
}
```
  对此，@Jessyan大神已经在他的文章中描述的很清楚了：
> RxCache并不是通过使用URL充当标识符来储存和获取缓存的.

>RxCache就是通过这两个对象加上上面CacheProviders接口中声明的方法名,组合起来一个标识符,通过这个标识符来存储和获取缓存

>标识符规则为:
>方法名 + \$d\$d\$d\$" + dynamicKey.dynamicKey + "\$g\$g\$g\$" + DynamicKeyGroup.group
>dynamicKey或DynamicKeyGroup为空时则返回空字符串,即什么都不传的标识符为:
>"方法名\$d\$d\$d\$\$g\$g\$g\$"

这些都说明，EvictRecord 、RetrieveRecord、SaveRecord 三个对象都持有内存层数据和持久层数据的引用，却又各自有各自的职责（数据的驱逐/获取/保存），分工明确，却都由上层的TwoLayersCache进行操作。

关于EvictRecord 、RetrieveRecord、SaveRecord这三个类，同样不细致化去讲，有兴趣的朋友或者有特殊需求可以去翻看源码。
 
 那么这个TwoLayersCache是由谁来控制呢，很显然，我们已经从图中得知，TwoLayersCache有且仅有一个类负责操作，那就是ProcessorProvidersBehaviour.
  
### 三.额外的操作支持

其实看到TwoLayersCache，我们认为Cache功能已经大致有所了解了，既然TwoLayersCache负责控制数据不同操作的Action，每个Action下都持有Memory和Persistence 的引用，为何还要再封装一层ProcessorProvidersBehaviour呢？

换句话说，ProcessorProvidersBehaviour除了TwoLayersCache的功能，还负责什么功能呢？

我们来看源码：
```
public final class ProcessorProvidersBehaviour implements ProcessorProviders {
  private final io.rx_cache2.internal.cache.TwoLayersCache twoLayersCache;
  private final Boolean useExpiredDataIfLoaderNotAvailable;
  private final GetDeepCopy getDeepCopy;
  private final Observable<Integer> oProcesses;
  private volatile Boolean hasProcessesEnded;

  @Inject public ProcessorProvidersBehaviour(
      io.rx_cache2.internal.cache.TwoLayersCache twoLayersCache,
      Boolean useExpiredDataIfLoaderNotAvailable,
      io.rx_cache2.internal.cache.EvictExpiredRecordsPersistence evictExpiredRecordsPersistence,
      GetDeepCopy getDeepCopy, io.rx_cache2.internal.migration.DoMigrations doMigrations) {
    this.hasProcessesEnded = false;
    this.twoLayersCache = twoLayersCache;//两层缓存
    this.useExpiredDataIfLoaderNotAvailable = useExpiredDataIfLoaderNotAvailable;//若Loader不可用，加载过期数据的配置
    this.getDeepCopy = getDeepCopy;//深拷贝
    this.oProcesses = startProcesses(doMigrations, evictExpiredRecordsPersistence);//数据迁移
  }
  //省略其他
}
``` 
 从图中也可以看到，ProcessorProvidersBehaviour 还负责其他功能，比如数据迁移，在DataLoader不可用状态下是否加载过期数据等等。
 
这些可选项都是RxCache提供的额外功能，具体使用方式也很简单，加上对应注解即可，详情请参考

 [《你不知道的Retrofit缓存库RxCache》](http://www.jianshu.com/p/b58ef6b0624b)

我们看到实际上ProcessorProvidersBehaviour 实现了ProcessorProviders接口，这个接口很简单：

```
public interface ProcessorProviders {

	//加载对应缓存数据
  <T> Observable<T> process(final ConfigProvider configProvider);

  //清除所有数据
  Observable<Void> evictAll();
}
```
 这个ConfigProvider中包含我们请求缓存相关所有配置：
```
public final class ConfigProvider {
  private final String providerKey;
  private final Boolean useExpiredDataIfNotLoaderAvailable;
  private final Long lifeTime;
  private final boolean requiredDetailedResponse;
  private final boolean expirable;
  private final boolean encrypted;
  private final String dynamicKey, dynamicKeyGroup;
  private final Observable loaderObservable;
  private final EvictProvider evictProvider;
	//省略其他
}
```
 
 现在我们可以说，如果我们只要有了ConfigProvider 和ProcessorProvidersBehaviour 对象，我们就能自己脑补出RxCache的工作流程了！

但是，问题来了。

### 四.我们一无所有

我们一无所有！

确实如此，看起来我们只需要ConfigProvider 和ProcessorProvidersBehaviour 对象，但是我们实际上需要很多很多依赖，才能创建这两个对象的实例：

* ConfigProvider依赖: 所有配置，包括dynamicKey， dynamicKeyGroup,是否加密，缓存失效时长，缓存文件名（providerkey）等等...
* ProcessorProvidersBehaviour依赖 ：两层缓存TwoLayersCache （它又包括Memory和Persistence的示例，有了他们才能创建对应的三个Action的子类，有了这三个子类才有TwoLayersCache 对象）、深拷贝、数据迁移等等.....

现在我们头大的发现，我们真的是一无所有，或者说，所需要实例化的依赖太多了！

现在，站在RxCache作者的角度去思考，接下来该怎么实现？

### 五、依赖注入：神兵利器Dagger2
### 六、动态代理：ProxyProviders

接下来就是核心实现方式了，不管你激动没有，反正我是激动了。
还记得笔者引用的一句话吗：

>  RxCache使用注解来为Retrofit配置缓存信息,内部使用动态代理和Dagger来实现。

关于动态代理和依赖注入框架Dagger2，不太了解的同学可以参考一下，本文不赘述：

[Java代理模式分析总结](http://blog.csdn.net/mq2553299/article/details/78370307)

[Android开发系列专栏：Dagger2详解](http://blog.csdn.net/column/details/17168.html)

我们看一下动态代理类ProxyProviders：

```
public final class ProxyProviders implements InvocationHandler {
  private final io.rx_cache2.internal.ProcessorProviders processorProviders;
  private final ProxyTranslator proxyTranslator;

  public ProxyProviders(RxCache.Builder builder, Class<?> providersClass) {
	//下面实例化的processorProviders就是ProcessorProvidersBehaviour！ 
    processorProviders = DaggerRxCacheComponent.builder()
        .rxCacheModule(new RxCacheModule(builder.getCacheDirectory(),
            builder.useExpiredDataIfLoaderNotAvailable(),
            builder.getMaxMBPersistenceCache(), getEncryptKey(providersClass),
            getMigrations(providersClass), builder.getJolyglot()))
        .build().providers();
	//ProxyTranslator内部提供了configProvider的实例化
    proxyTranslator = new ProxyTranslator();
  }
}
```
 对于依赖注入和动态代理不陌生的朋友们，看到段代码应该就明白了，通过依赖注入和注解，取得缓存数据所需要的依赖配置。

而缓存数据的处理，则交给依赖动态代理生成的代理类去做。
 
 至于这个代理类的示例是在哪里使用的呢？

```
public final class RxCache {
  private final Builder builder;
  private ProxyProviders proxyProviders;//动态代理类

  private RxCache(Builder builder) {
    this.builder = builder;
  }
	//就是它，获取代理类的真实对象
  public <T> T using(final Class<T> classProviders) {
    proxyProviders = new ProxyProviders(builder, classProviders);

    return (T) Proxy.newProxyInstance(
        classProviders.getClassLoader(),
        new Class<?>[] {classProviders},
        proxyProviders);
  }
  //省略其他代码和RxCache建造者模式相关配置
 }
```
 在我们项目中的代码中，当我们调用下面这行代码：
```
new RxCache.Builder()
                 .persistence(BaseApplication.getApplication().getExternalCacheDir(), new GsonSpeaker())
                    .using(UserCacheProviders.class);
```
 实际上我们就已经获得了该UserCacheProviders的对象，并可以进行接下来缓存数据的操作了。
 
 至此，RxCache的原理分析也基本告一段落。

## 感谢

一篇文章下来，已是深夜，但是跟着代码，回顾了[RxCache](https://github.com/VictorAlbertos/RxCache) 这个库的设计思路，一条线捋下来，亦是酣畅淋漓。

最后还要感谢[@Jessyan](http://www.jianshu.com/u/1d0c0bc634db) 的文章和图，说实话，本文的一些思路和结论也是从他的文章中思索得来，没有开源社区一些先驱者的开拓，后者的我们亦不会如此顺利得到结果。

还要感谢[@VictorAlbertos](https://github.com/VictorAlbertos/)的这个库[RxCache](https://github.com/VictorAlbertos/RxCache),让我们能够看到，优秀的框架设计所能给我们带来的魅力。
