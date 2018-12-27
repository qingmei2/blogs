## 发现

最近在研究[VictorAlbertos](https://github.com/VictorAlbertos)大神的[RxCache](https://github.com/VictorAlbertos/RxCache)库时，发现了他的另外一个库:[RxActivityResult](https://github.com/VictorAlbertos/RxActivityResult)。

这个库顾名思义，就是将Android中的startActivityForResult()事件转换为Rx事件流，我花了一点时间看了看并且去尝试了一下，发现效果比想象中的还要好。

今天笔者简单介绍下这个库的使用，并且分析下这个库的价值。
 
## 常规写法
 
 我们首先来看常规代码我们如何实现Activity之间的数据传递：
 
我们创建两个传递数据用的Activity：

MainResultActivity(Result的请求者)  和 SecondResultActivity（Result的发送者）。
 
### MainResultActivity

```java
public class MainResultActivity extends AppCompatActivity {
	
	//我们需要自己写一个常量作为requestCode，在请求result时传递进去
    public static final int REQUEST_CODE_NORMAL = 100;
	
	//我们省略其他无关紧要代码
	//打开新的界面，请求result
    public void startByNormal() {
        startActivityForResult(new Intent(this,
                        SecondResultActivity.class),
                REQUEST_CODE_NORMAL);
    }
	
	//获得Result数据并处理
    @Override
    protected void onActivityResult(int requestCode, int resultCode, Intent data) {
        super.onActivityResult(requestCode, resultCode, data);
        if (requestCode == REQUEST_CODE_NORMAL) {
            showResultIntentData(data);
        }
    }
	
	//处理Result数据并展示
    public void showResultIntentData(Intent data) {
        String content = data.getStringExtra("content");
        tvResult.setText("传回来的内容：");
        tvResult.append(content);
    }
}

```

### SecondResultActivity

```
public class SecondResultActivity extends AppCompatActivity {   

	//我们省略其他无关紧要代码
	//发送Result数据给请求方，然后finish（）
	 public void commitResult() {
        Intent intent = new Intent(this,MainResultActivity.class);
        intent.putExtra("content",etContent.getText().toString());
        setResult(1,intent);
        finish();
    }
}
```
好的，效果基本可以实现，我们其实可以发现，本身实现难度也不大，代码量也不是很多。

我们来看一下Rx和其操作符的威力。

## RxActivityResult

### 添加依赖

在Project级别的build.gradle中添加：
 
```groovy
allprojects {
    repositories {
        jcenter()
        maven { url "https://jitpack.io" }
    }
}
```
在module级别的build.gradle中添加：
```groovy
dependencies {
    compile 'com.github.VictorAlbertos:RxActivityResult:0.4.5-2.x'
    compile 'io.reactivex.rxjava2:rxjava:2.0.5'
}
```

### MainResultActivity

```

public class MainResultActivity extends AppCompatActivity {
	
  //这是一个Button点击后调用的方法：
  //打开新的界面，请求result,并进行数据结果的处理
  public void startByRxActivityResult() {
        RxActivityResult.on(this)
                .startIntent(new Intent(this, SecondResultActivity.class))//请求result
                .map(result -> result.data())//对result的处理，转换为intent
                .subscribe(intent -> showResultIntentData(intent));//处理数据结果
  }
  
  //处理数据结果
  public void showResultIntentData(Intent data) {
      String content = data.getStringExtra("content");
      tvResult.setText("传回来的内容：");
      tvResult.append(content);
  }
}
```
SecondResultActivity中的处理不变。
  

可以看到，startByRxActivityResult()方法中，一行代码的链式调用即可完成：

> ① 打开新的界面，请求result
> ② 进行数据结果的处理
> ③ 不需要自己实现一个常量作为requestCode，并在请求result时传递进去

### onNext中的返回值 Result：

 在subscribe()的onNext()回调中返回的Result对象是作者封装的一个类，我们可以从中取得很多东西：

```
public class Result<T> {
    private final T targetUI;//订阅事件发生时所在的容器，本文中为MainResultActivity.
    private final int resultCode;//resultCode
    private final int requestCode;//requestCode
    private final Intent data;//存储数据的Intent对象

    public Result(T targetUI, int requestCode, int resultCode, Intent data) {
        this.targetUI = targetUI;
        this.resultCode = resultCode;
        this.requestCode = requestCode;
        this.data = data;
    }

    public int requestCode() {
        return this.requestCode;
    }

    public int resultCode() {
        return this.resultCode;
    }

    public Intent data() {
        return this.data;
    }

    public T targetUI() {
        return this.targetUI;
    }
}

```
 
 原来，Result和正常方式下，Activity的onActivityResult()是一样的，只不过加了一层封装而已：

```
//获得Result数据并处理
    @Override
    protected void onActivityResult(int requestCode, int resultCode, Intent data) {
        super.onActivityResult(requestCode, resultCode, data);
        if (requestCode == REQUEST_CODE_NORMAL) {
            showResultIntentData(data);
        }
    }
```
 
## 闪光点

 事实上，如果仅仅只有这些功能，这个库也仅仅是鸡肋，但是配合RxJava丰富的操作符和众多的Rx拓展库一起使用，佐以Java8的lambda表达式和方法引用，我们可以高度定制很多东西，比如上面的代码：

```
 //这是一个Button点击后调用的方法：
  //打开新的界面，请求result,并进行数据结果的处理
  public void startByRxActivityResult() {
        RxActivityResult.on(this)
                .startIntent(new Intent(this, SecondResultActivity.class))//请求result
                .map(result -> result.data())//对result的处理，转换为intent
                .subscribe(intent -> showResultIntentData(intent));//处理数据结果
  }

```
 
 我们假设，如果这是一个Button的点击事件，我们其实代码中应该在Activity的某处地方实现一行代码：

> button.setOnClickListener(v -> startByRxActivityResult());
 
 如果配合RxJava全家桶，我们其实可以这么实现：

```java

//设置点击监听
private void initView(){
	RxView.clicks(button)	//RxBinding设置点击事件
          .throttleFirst(500, TimeUnit.MILLISECONDS)//防抖动
          .map(v -> new Intent(this, SecondResultActivity.class))//View转换为Intent
          .flatMap(intent -> RxActivityResult.on(this)
                  .startIntent(intent))//转换为startActivityForResult事件
          .map(result -> result.data())//对result的处理，转换为intent
          .subscribe(intent -> showResultIntentData(intent));//处理数据结果
}

//处理数据结果
public void showResultIntentData(Intent data) {
    String content = data.getStringExtra("content");
    tvResult.setText("传回来的内容：");
    tvResult.append(content);
}
```
 
 其实2个方法也很多余，于是我们变成这样：

```
//设置点击监听
private void initView(){
	RxView.clicks(button)	//RxBinding设置点击事件
          .throttleFirst(500, TimeUnit.MILLISECONDS)//防抖动
          .map(v -> new Intent(this, SecondResultActivity.class))//View转换为Intent
          .flatMap(intent -> RxActivityResult.on(this)
                  .startIntent(intent))//转换为startActivityForResult事件
          .map(result -> result.data())//对result的处理，转换为intent
          .map(intent -> intent.getStringExtra("content"))//获取要显示的内容
          .map(content -> "传回来的内容："+content)
          .subscribe(content -> tvResult.setText(content));//设置显示
}
```
然后你说lambda表达式中的参数也很多余，于是我们变成方法引用：

```
//设置点击监听
private void initView(){
	RxView.clicks(button)	
          .throttleFirst(500, TimeUnit.MILLISECONDS)
          .map(v -> new Intent(this, SecondResultActivity.class))
          .flatMap(intent -> RxActivityResult.on(this)
                  .startIntent(intent))
          .map(Result::data)
          .map(intent -> intent.getStringExtra("content"))
          .map(content -> "传回来的内容："+content)
          .subscribe(tvResult::setText);
}
```
 大功告成，一个链式调用，从点击监听，到请求其他Activity数据，数据获取，数据处理，数据展示，全部搞定，最大的好处是：代码干净，一条链下来，处理的逻辑十分清晰。
 
## 小结

RxJava强大在于其操作符，如果我们能够合理利用操作符，我们的代码能够变得更加简洁，当然前提是需要花费一些时间成本去学习，但是绝对值得。

RxActivityResult这个库并非一定需要我们在项目中去使用，它的源码也有很大一部分的学习价值（设计的思想和RxJava操作符的合理使用），有机会笔者会尝试从源码的角度分析这个库的价值。
 
 最后附上该库的官方网址，以及本文demo对应的源码：
 
[VictorAlbertos_RxActivityResult 官方传送门](https://github.com/VictorAlbertos/RxActivityResult)

[本文的Demo示例源码传送门](https://github.com/qingmei2/RxFamilyUsage-Android)
