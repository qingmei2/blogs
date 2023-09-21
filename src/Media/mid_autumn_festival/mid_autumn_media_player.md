## 概述

前几天发现QQ音乐有个好玩的功能，为用户提供了多种 **播放器主题**，其中 **原神** 的主题让我眼前一亮：

<center class="half">
    <img src="https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/1101ad89954440278959a7e07cd1df01~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=1080&h=2400&s=1315133&e=jpg&b=badbe0" width="160" height="320" alt="转存失败，建议直接上传图片文件">
</center>

当然，诸如 **换肤、主题** 类的功能已经屡见不鲜，但这类沉浸式播放器的听歌体验确实不错。

见猎心喜，正好中秋马上就到，我也尝试整个 **中秋主题音乐播放器** 试试水。

整体思路有2点：

首先是技术方面，纯粹的 `ImageView` 图层堆砌来实现，**渲染效率太低**，`OpenGL` 是一个不错的技术方案（`QQ`应该也是这么实现的），顺便复习下图形学的知识。

其次是玩法上，干脆在基础的功能上加一些 **更好玩的**，比如为播放页设计多个图层，通过陀螺仪+图层联动实现的 **裸眼3D** 的视觉效果，边听歌边玩。

说了这么多，最后效果如下所示，左侧展示录屏效果，右侧是裸眼3D效果：

<center class="half">
    <img src="https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/a8adc699bc854a019a681c310ace3a60~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=364&h=749&s=1411546&e=gif&f=55&b=3b3962" width="310" height="640">
    <img src="https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/7600189727b84e6ea453e2db6570f218~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=348&h=627&s=10801585&e=gif&f=90&b=4d54b5" width="360" height="640" >
</center>

## 具体实现

### 1. 裸眼3D原理

2年前自如的 [《自如客APP裸眼3D效果的实现》](https://juejin.cn/post/6989227733410644005) 一文引发了社区的热烈讨论和实践，本着不重复造轮子的原则，这里简单对原理介绍，感兴趣的读者可参考上述链接。

裸眼 `3D` 效果的本质是——将整个图片结构分为 `3` 层：上层、中层、以及底层。在手机左右上下旋转时，上层和底层的图片呈相反的方向进行移动，中层则不动，在视觉上给人一种 `3D` 的感觉：

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/9909cd5efd0540c8985e817965179eed~tplv-k3u1fbpfcp-watermark.awebp)

本文的效果是由以下四张图，由底至顶，依次绘制而成的：

| 背景                                                                                                                                                                        | 中景月亮                                                                                                                                                                             | 中景歌曲封面                                                                                                                                                                          | 前景                                                                                                                                                                                |
| ------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| ![](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/b513b7d85cfb4a0dbca68804b6ba8b0b~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=1080\&h=1528\&s=330\&e=png\&b=3a3b64) | ![](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/bba3381129de437591f9e3cfcd6fe8e6~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=1080\&h=1528\&s=29845\&e=png\&a=1\&b=f8e6c3) | ![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/b90efccdda814909ac612091b33739d7~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=800\&h=800\&s=309066\&e=png\&a=1\&b=e3d2c5) | ![](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/4cd6ae70ae96469a910b5f9342e50d4a~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=1080\&h=1528\&s=200856\&e=png\&a=1\&b=fdf6f4) |

接下来，如何感应手机的旋转状态，并将4层图片进行对应的移动呢？当然是使用设备自身提供的 **传感器** 了，通过传感器不断回调获取设备的旋转状态，对 `UI` 进行对应地渲染即可。

### 2. 为何选择 OpenGL

`GPU` 更适合图形、图像的处理，裸眼3D效果中有大量的 **旋转**、**缩放** 和 **位移** 操作，都可在 `java` 层通过一个 **矩阵** 对几何变换进行描述，通过 `shader` 小程序中交给 `GPU` 处理 ——理论上 `OpenGL` 的渲染性能比原生的 `ImageView` 更好。

借助`OpenGL`的`API`，渲染性能也符合预期，打开 **布局边界** 和 **GPU过渡绘制** 选项后，播放页渲染性能也依然稳定，更不会增加布局层级的复杂度，直接证明了该方案 **具备应用到实际生产项目的可行性**：

<center class="half">
    <img src="https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/6d6c36b1feb342029d8b6d12a9ffe256~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=364&h=750&s=3009507&e=gif&f=38&b=e76781" width="310" height="640" alt="">
</center>

### 3.代码实现

> 本文重点是描述 OpenGL 绘制时的思路描述，因此下文仅展示部分核心代码，对具体实现感兴趣的读者可参考文末的链接。

#### 3.1 绘制静态图片

首先需要将4张图片（[图片素材来源](https://www.freepik.com/search?format=search\&query=%E4%B8%AD%E7%A7%8B\&type=psd#)）依次进行静态绘制，这里涉及大量 `OpenGL API` 的使用，不熟悉的读可略读本小节，以捋清思路为主。

首先看一下顶点和片元着色器的 `shader` 代码，其定义了图像纹理是如何在`GPU`中处理渲染的：

```java
// 顶点着色器代码
// 顶点坐标
attribute vec4 av_Position;
// 纹理坐标
attribute vec2 af_Position;
uniform mat4 u_Matrix;
varying vec2 v_texPo;

void main() {
    v_texPo = af_Position;
    gl_Position =  u_Matrix * av_Position;
}
```

```java
// 片元着色器代码
precision mediump float;
// 纹理坐标
varying vec2 v_texPo;
uniform sampler2D sTexture;
void main() {
    gl_FragColor=texture2D(sTexture, v_texPo);
}

```

定义好了 `Shader` ，接下来在 `GLSurfaceView` (可以理解为 `OpenGL` 中的画布) 创建时，初始化`Shader`小程序，并将图像纹理依次加载到`GPU`中：

```java
public class ZQRenderer implements GLSurfaceView.Renderer {
  
  @Override
  public void onSurfaceCreated(GL10 gl, EGLConfig config) {
      // 1.加载shader小程序
      mProgram = loadShaderWithResource(mContext, R.raw.projection_vertex_shader, R.raw.projection_fragment_shader);

      //...
      
      // 2. 依次将切图纹理传入GPU
      this.texImageInner(R.drawable.icon_player_bg, mBackTextureId);
      this.texImageInner(R.drawable.icon_player_moon, mMidTextureId);
      this.texImageInner(R.drawable.icon_album_cover_nocturne, mCoverTextureId);
      this.texImageInner(R.drawable.icon_player_text, mFrontTextureId);
  }
}
```

接下来是定义视口以及投影矩阵，因为切图的比例各不相同，为了保证视觉效果，需要针对不同层级的图片，设置不同的正交投影策略。

```java
public class ZQRenderer implements GLSurfaceView.Renderer {
  
    @Override
    public void onSurfaceChanged(GL10 gl, int width, int height) {
        //设置大小位置
        GLES20.glViewport(0, 0, width, height);

        Matrix.setIdentityM(mBgProjectionMatrix, 0);
        Matrix.setIdentityM(mMoonProjectionMatrix, 0);
        Matrix.setIdentityM(mCoverProjectionMatrix, 0);

        // 计算宽高比
        boolean isVertical = width < height;
        float screenRatio = (float) width / (float) height;

        // 设置投影矩阵
        // 1.深色背景图的投影矩阵，只需要铺全屏

        // 2.月亮和装饰图的投影矩阵
        float ratio = (float) 1080 / (float) 1528;
        if (isVertical) {
            Matrix.orthoM(mMoonProjectionMatrix, 0, -1f, 1f, -1f / ratio, 1f / ratio, -1f, 1f);
        } else {
            Matrix.orthoM(mMoonProjectionMatrix, 0, -ratio, ratio, -1f, 1f, -1f, 1f);
        }

        // 3.歌曲封面图投影矩阵
        if (isVertical) {
            Matrix.orthoM(mCoverProjectionMatrix, 0, -1f, 1f, -1f / screenRatio, 1f / screenRatio, -1f, 1f);
        } else {
            Matrix.orthoM(mCoverProjectionMatrix, 0, -screenRatio, screenRatio, -1f, 1f, -1f, 1f);
        }
    }
}
```

最后就是 **绘制**，读者需要理解，对于4层图像的渲染，其逻辑是基本一致的，差异仅仅有2点：**图像本身不同** 以及 **图像的几何变换不同**。

```java
public class ZQRenderer implements GLSurfaceView.Renderer {
  
  @Override
     public void onDrawFrame(GL10 gl) {
         GLES20.glClear(GLES20.GL_COLOR_BUFFER_BIT);
         GLES20.glClearColor(0.0f, 0.0f, 0.0f, 1.0f);
 
         GLES20.glUseProgram(mProgram);
 
         this.updateMatrix();
 
         this.drawLayerInner(mBackTextureId, mTextureBuffer, mBackMatrix);  // 画背景
         this.drawLayerInner(mMidTextureId, mTextureBuffer, mMoonMatrix);   // 画月亮
         this.drawLayerInner(mCoverTextureId, mTextureBuffer, mCoverMatrix);  // 画封面
         this.drawLayerInner(mFrontTextureId, mTextureBuffer, mFrontMatrix);  // 画前景装饰
     }
 
     private void texImageInner(@DrawableRes int drawableRes, int textureId) {
         //绑定纹理
         GLES20.glBindTexture(GLES20.GL_TEXTURE_2D, textureId);
         //环绕方式
         GLES20.glTexParameteri(GLES20.GL_TEXTURE_2D, GLES20.GL_TEXTURE_WRAP_S, GLES20.GL_REPEAT);
         GLES20.glTexParameteri(GLES20.GL_TEXTURE_2D, GLES20.GL_TEXTURE_WRAP_T, GLES20.GL_REPEAT);
         //过滤方式
         GLES20.glTexParameteri(GLES20.GL_TEXTURE_2D, GLES20.GL_TEXTURE_MIN_FILTER, GLES20.GL_LINEAR);
         GLES20.glTexParameteri(GLES20.GL_TEXTURE_2D, GLES20.GL_TEXTURE_MAG_FILTER, GLES20.GL_LINEAR);
 
         GLES20.glEnable(GLES20.GL_BLEND);
         GLES20.glBlendFunc(GL_SRC_ALPHA, GL_ONE_MINUS_SRC_ALPHA);
 
         Bitmap bitmap = BitmapFactory.decodeResource(mContext.getResources(), drawableRes);
         GLUtils.texImage2D(GLES20.GL_TEXTURE_2D, 0, bitmap, 0);
         bitmap.recycle();
     }
}
```

现在我们完成了图像的 **静态绘制**，接下来我们需要接入 **传感器**，并引入不同层级图片各自的几何变换， **让图片动起来**。

### 2. 让图片动起来

首先我们需要对 `Android` 平台上的传感器进行注册，监听手机的旋转状态，并拿到手机 `xy` 轴的旋转角度。

```java
// 2.1 注册传感器
mSensorManager = (SensorManager) context.getSystemService(Context.SENSOR_SERVICE);
mAcceleSensor = mSensorManager.getDefaultSensor(Sensor.TYPE_ACCELEROMETER);
mMagneticSensor = mSensorManager.getDefaultSensor(Sensor.TYPE_MAGNETIC_FIELD);
mSensorManager.registerListener(mSensorEventListener, mAcceleSensor, SensorManager.SENSOR_DELAY_GAME);
mSensorManager.registerListener(mSensorEventListener, mMagneticSensor, SensorManager.SENSOR_DELAY_GAME);

// 2.2 不断接受旋转状态
private final SensorEventListener mSensorEventListener = new SensorEventListener() {
    @Override
    public void onSensorChanged(SensorEvent event) {
        // ... 省略具体代码
        float[] values = new float[3];
        float[] R = new float[9];
        SensorManager.getRotationMatrix(R, null, mAcceleValues, mMageneticValues);
        SensorManager.getOrientation(R, values);
        // x轴的偏转角度
        float degreeX = (float) Math.toDegrees(values[1]);
        // y轴的偏转角度
        float degreeY = (float) Math.toDegrees(values[2]);
        // z轴的偏转角度
        float degreeZ = (float) Math.toDegrees(values[0]);
        
        // 拿到 xy 轴的旋转角度，进行矩阵变换
        updateMatrix(degreeX, degreeY);
    }
};
```

注意，因为我们只需控制图像的左右和上下移动，因此，我们只需关注设备本身 `x` 轴和 `y` 轴的偏转角度。但如果将图片直接进行位移操作，将会因为位移后图像的另一侧没有纹理数据，导致渲染结果有黑边现象，为了避免这个问题，我们需要将图像默认从中心点进行放大，保证图像移动的过程中，不会超出自身的边界。

也就是说，我们一开始进入时，看到的肯定只是图片的部分区域。给每一个图层设置 `scale`，将图片进行放大。显示窗口是固定的，那么一开始只能看到图片的正中位置。（中层可以不用，因为中层本身是不移动的，所以也不必放大)

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/4d4e6fadafc746b0a3fbf0da9695a6eb~tplv-k3u1fbpfcp-watermark.awebp)

> 这里的处理参考自 [Nayuta](https://juejin.cn/user/4309694831660711) 的 [这篇文章](https://juejin.cn/post/6991409083765129229#heading-4)，内部已经将思路阐述的非常清晰，强烈建议读者进行阅读。

明白了这一点，我们就能理解，裸眼 `3D` 的效果实际上就是对  **不同层级的图像** 进行 **缩放** 和 **位移** 的变换，下面是分别获取几何变换的代码：

```java
public class ZQRenderer implements GLSurfaceView.Renderer {
  
  private float[] mBgProjectionMatrix = new float[16];
  private float[] mMoonProjectionMatrix = new float[16];
  private float[] mCoverProjectionMatrix = new float[16];

  private float[] mBackMatrix = new float[16];
  private float[] mMoonMatrix = new float[16];
  private float[] mCoverMatrix = new float[16];
  private float[] mFrontMatrix = new float[16];

  // 封面图旋转一圈的时间，单位秒.
  private static final long ROTATE_TIME = 20L;
  public static final long DELAY_INTERVAL = 1000 / (360 / ROTATE_TIME);

  /**
   * 陀螺仪数据回调，更新各个层级的变换矩阵.
   *
   * @param degreeX x轴旋转角度，图片应该上下移动
   * @param degreeY y轴旋转角度，图片应该左右移动
   */
   private void updateMatrix() {
       //  ----------  背景-蓝色底图  ----------
       Matrix.setIdentityM(mBackMatrix, 0);
       // 1.最大位移量
       float maxTransXY = MAX_VISIBLE_SIDE_BACKGROUND - 1f;
       // 2.本次的位移量
       float transX = ((maxTransXY) / MAX_TRANS_DEGREE_Y) * -mCurDegreeY;
       float transY = ((maxTransXY) / MAX_TRANS_DEGREE_X) * -mCurDegreeX;
       float[] backMatrix = new float[16];
       // 蓝色底图的投影矩阵，需要铺展全屏.
       Matrix.setIdentityM(mBgProjectionMatrix, 0);
       Matrix.setIdentityM(backMatrix, 0);
       Matrix.translateM(backMatrix, 0, transX, transY, 0f);                    // 2.平移
       Matrix.scaleM(backMatrix, 0, SCALE_BACK_GROUND, SCALE_BACK_GROUND, 1f);  // 1.缩放
       Matrix.multiplyMM(mBackMatrix, 0, mBgProjectionMatrix, 0, backMatrix, 0);  // 3.正交投影

       //  ----------  背景 -月亮  ----------
       Matrix.setIdentityM(mMoonMatrix, 0);
       float[] midMatrix = new float[16];
       Matrix.setIdentityM(midMatrix, 0);
//        Matrix.translateM(midMatrix, 0, transX, transY, 0f);                      // 2.平移，这行注释解开后，手机摇一摇，封面图和月亮也会有位移偏差.
       Matrix.scaleM(midMatrix, 0, SCALE_MOON_GROUND, SCALE_MOON_GROUND, 1.0f);  // 1.缩放
       Matrix.multiplyMM(mMoonMatrix, 0, mMoonProjectionMatrix, 0, midMatrix, 0);  // 3.正交投影

       // ---------  中景-歌曲封面  ----------
       Matrix.setIdentityM(mCoverMatrix, 0);
       float[] rotateMatrix = new float[16];
       float[] tranAndScale = new float[16];
       float[] coverMatrix = new float[16];

       Matrix.setIdentityM(rotateMatrix, 0);
       Matrix.setIdentityM(tranAndScale, 0);
       Matrix.setIdentityM(coverMatrix, 0);

       Matrix.scaleM(tranAndScale, 0, 0.565f, 0.58f, 1.0f);                   // 3.缩放,这里的缩放参数是开发时，即时调整的，保证歌曲封面和月亮的大小一致
       Matrix.translateM(tranAndScale, 0, 0.05f, 1.41f, 0f);                 // 2.平移,这里的位移参数是开发时，即时调整的，保证歌曲封面和月亮的center位置在一起

       Matrix.setRotateM(rotateMatrix, 0, 360 - mCoverDegree, 0.0f, 0.0f, 1.0f);    // 1.旋转，顺时针

       Matrix.multiplyMM(coverMatrix, 0, tranAndScale, 0, rotateMatrix, 0);
       Matrix.multiplyMM(mCoverMatrix, 0, mCoverProjectionMatrix, 0, coverMatrix, 0);  // 4.正交投影

       //  ----------  前景-装饰  ----------
       Matrix.setIdentityM(mFrontMatrix, 0);
       // 1.最大位移量
       maxTransXY = MAX_VISIBLE_SIDE_FOREGROUND - 1f;
       // 2.本次的位移量
       transX = ((maxTransXY) / MAX_TRANS_DEGREE_Y) * -mCurDegreeY;
       transY = ((maxTransXY) / MAX_TRANS_DEGREE_X) * -mCurDegreeX;
       float[] frontMatrix = new float[16];
       Matrix.setIdentityM(frontMatrix, 0);
       Matrix.translateM(frontMatrix, 0, -transX, -transY, 0f);         // 2.平移
       Matrix.scaleM(frontMatrix, 0, SCALE_FORE_GROUND, SCALE_FORE_GROUND, 1f);    // 1.缩放
       Matrix.multiplyMM(mFrontMatrix, 0, mMoonProjectionMatrix, 0, frontMatrix, 0);  // 3.正交投影
   }
}
```

背景、月亮、前景都很简单，只有 **中景的歌曲封面** 麻烦一些，首先歌曲封面要伴着歌曲进度做 **旋转动画**，其次，由于图片素材尺寸的原因，中心点要 **位移** 到和月亮相同的位置，最后 **缩放** 到和月亮一样的大小完成重合。

## 小结

现在，我们完成了图示效果的开发。

限于篇幅，文中代码以捋清思路为主，部分细节(如 `Handler` 不断发消息实现旋转动画、添加 **低通滤波器** 防止抖动等)没展示出来，感兴趣的小伙伴可以点击 [这里](https://github.com/qingmei2/OpenGL-demo/blob/master/app/src/main/java/com/github/qingmei2/opengl_demo/d_zhongqiu/ZQRenderer.java) 查看源码。

## 关于我

Hello，我是 [却把清梅嗅](https://github.com/qingmei2) ，如果您觉得文章对您有价值，欢迎 ❤️，也欢迎关注我的 [博客](https://blog.csdn.net/mq2553299) 或者 [GitHub](https://github.com/qingmei2)。

*   [我的Android学习体系](https://github.com/qingmei2/blogs)
*   [关于文章纠错](https://github.com/qingmei2/blogs/blob/main/error_collection.md)
*   [关于知识付费](https://github.com/qingmei2/blogs/blob/main/appreciation.md)
*   [关于《反思》系列](https://github.com/qingmei2/blogs/blob/main/src/%E5%8F%8D%E6%80%9D%E7%B3%BB%E5%88%97/thinking_in_android_index.md)
