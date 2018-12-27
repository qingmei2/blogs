## 概述
 **Camera** 可能是接下来个人想深入学习的课题，准备新起一个系列，从个人的角度总结阐述自己对于 **Android Camera** 的研究过程，希望也能够对其他想学习 **Camera** 的同学一些帮助。

> 本小节内容为 **Android Camera 官方文档** 的精要翻译，原文请参考：
> * [Android Camera: Camera API](https://developer.android.com/guide/topics/media/camera#preview-layout)  

# 正文

Android Framework 包括对设备上可用的摄像头和摄像头功能的支持，以达到在应用程序中 **拍照和录制视频** 的目的。 本文档讨论了一种快速，简单的拍照和录制视频的实现方法，并概述了为用户创建 **自定义相机体验** 的高阶使用说明。

> **请注意**：本文示例代码使用的是已弃用的`Camera`类。 Google官方建议使用新的API `Camera2`，它适用于Android 5.0（API级别21）或更高版本——但是国内的Rom似乎对`Camera2`的支持都不是非常友好，并且仅支持API21以上的Android系统，因此`Camera`依然有研究学习的价值。

## 一、注意事项
您的APP在Android设备上使用`Camera`相关API之前，应该注意的：

#### 1.摄像头要求 
 您应该在`Manifest.xml`中声明`Camera`的使用，以保证安装App的Android设备都有`Camera`的硬件支持。

#### 2.优先考虑使用系统相机
如果您只想**拍摄快照**或**录制视频**，而不会使用`Camera`的其它API,请考虑使用系统相机。

#### 3.前台服务要求 
在Android 9（API级别28）及更高版本中，在后台运行的应用程序无法访问摄像头。因此，您应该在应用程序位于前台或作为前台服务的一部分时使用相机。

#### 4.存储 

您的应用程序生成的 图像或视频是否仅对您的应用程序 **可见或共享**，以便其他应用程序（比如 **相册** 或者其他应用）可以使用它们？即使您的应用程序已卸载，您是否希望图片和视频同样被删除？

## 二、基础
Android FrameWork 支持通过`Camera2` API或相机`Intent`捕获图像和视频。 以下是相关官方API文档（注意：**下面链接请自备梯子**）：

* [android.hardware.camera2](https://developer.android.com/reference/android/hardware/camera2/package-summary.html)

控制Android设备摄像头的主要API。 在构建相机应用程序时，它可用于拍摄照片或视频。

* [Camera](https://developer.android.com/reference/android/hardware/Camera.html)

用于控制设备相机，较旧的、已弃用的API。

* [SurfaceView](https://developer.android.com/reference/android/view/SurfaceView.html)

相机捕获的图像都会在展示在上面**预览**。

* [MediaRecorder](https://developer.android.com/reference/android/media/MediaRecorder.html)

用于录制来自摄像机的视频。

* [Intent](https://developer.android.com/reference/android/content/Intent.html)

直接通过Intent访问系统相机，而无需调用`Camera`对象。

## 三、使用系统相机

如果您只想**拍摄快照**或**录制视频**，而不会使用`Camera`的其它API,请优先考虑使用系统相机，详细实现请参考[Taking Photos Simply](https://developer.android.com/training/camera/photobasics.html) 或者 [Recording Videos Simply](https://developer.android.com/training/camera/videobasics.html).

上面2个link对应的中文博客请参考这篇：

> [Android Camera 系列（一）拍照和录制视频](https://www.imooc.com/article/253961)

## 四、构建相机应用

创建一个自定义的相机步骤如下：

* 检测并获取相机
* 创建一个预览类：创建一个继承自`SurfaceView`的类并实现`SurfaceHolder`接口。这个类预览相机的实时图像。
* 构建一个预览界面：创建完相机预览类以后，创建一个布局以展示预览。
* 增加监听：增加对应的监听，以操作`Camera`的行为，比如点击按钮拍照。
* 采集并保存文件 ：将拍照图片或视频保存。
* 释放相机资源 -使用完相机以后，**必须正确释放相机资源，以供其他应用继续使用**。

### 1.检测相机硬件

```kotlin
/** 检查相机是否可用 **/
fun cameraHardwareAvailable() =
        packageManager.hasSystemFeature(PackageManager.FEATURE_CAMERA)
```

### 2.访问相机

如果您确定Android设备有摄像头，则必须通过获取`Camera`实例来使用相关API（除非您使用`Intent`访问系统相机）。

要访问主摄像头，请使用Camera.open（）方法并确保捕获任何异常，如下面的代码所示：

```kotlin
/** A safe way to get an instance of the Camera object. */
fun getCameraInstance(): Camera? {
    return try {
        Camera.open() // attempt to get a Camera instance
    } catch (e: Exception) {
        // Camera is not available (in use or does not exist)
        null // returns null if camera is unavailable
    }
}
```
> 警告：使用`Camera.open()`时，请务必检查异常。 如果相机正在使用或不存在，则无法检查异常将导致崩溃。

在 Android 2.3 以上，你可以使用`Camera.open(int) `获取特定的相机，它将在有多个相机的设备上获取**第一个后置摄像头**。

### 3.检查相机特性

得到`Camera`实例以后，你可以使用`Camera.getParameters() `方法获取关于相机的更多**属性参数**的`Parameters`对象。

### 4.创建一个预览类

以下的示例代码展示了如何创建一个基本的相机预览类：

```kotlin
/** A basic Camera preview class */
class CameraPreview(
        context: Context,
        private val mCamera: Camera
) : SurfaceView(context), SurfaceHolder.Callback {

    private val mHolder: SurfaceHolder = holder.apply {
        // Install a SurfaceHolder.Callback so we get notified when the
        // underlying surface is created and destroyed.
        addCallback(this@CameraPreview)
        // deprecated setting, but required on Android versions prior to 3.0
        setType(SurfaceHolder.SURFACE_TYPE_PUSH_BUFFERS)
    }

    override fun surfaceCreated(holder: SurfaceHolder) {
        // The Surface has been created, now tell the camera where to draw the preview.
        mCamera.apply {
            try {
                setPreviewDisplay(holder)
                startPreview()
            } catch (e: IOException) {
                Log.d(TAG, "Error setting camera preview: ${e.message}")
            }
        }
    }

    override fun surfaceDestroyed(holder: SurfaceHolder) {
        // empty. Take care of releasing the Camera preview in your activity.
    }

    override fun surfaceChanged(holder: SurfaceHolder, format: Int, w: Int, h: Int) {
        // If your preview can change or rotate, take care of those events here.
        // Make sure to stop the preview before resizing or reformatting it.
        if (mHolder.surface == null) {
            // preview surface does not exist
            return
        }

        // stop preview before making changes
        try {
            mCamera.stopPreview()
        } catch (e: Exception) {
            // ignore: tried to stop a non-existent preview
        }

        // set preview size and make any resize, rotate or
        // reformatting changes here

        // start preview with new settings
        mCamera.apply {
            try {
                setPreviewDisplay(mHolder)
                startPreview()
            } catch (e: Exception) {
                Log.d(TAG, "Error starting camera preview: ${e.message}")
            }
        }
    }
}
```

如果要为相机的预览设置特定大小，请在`surfaceChanged()`方法中进行设置，如上面的注释中所述。 设置预览大小时，必须使用`getSupportedPreviewSizes()`中的值。不要在`setPreviewSize()`方法中随意地设置值。

> 注意：随着Android 7.0（API24）及更高级别中的**多窗口功能**的引入，即使在调用`setDisplayOrientation()`之后，您也不能再认为预览的宽高比与您的活动相同。 根据窗口大小和宽高比，您可能需要使用特定布局将宽相机预览适合纵向布局，反之亦然。

### 5.将相机预览展示出来

以下代码展示如何进行预览。其中`FrameLayout`用来作为相机预览类的布局容器：

```xml
<?xml version="1.0" encoding="utf-8"?>
<LinearLayout xmlns:android="http://schemas.android.com/apk/res/android"
    android:orientation="horizontal"
    android:layout_width="fill_parent"
    android:layout_height="fill_parent"
    >
  <FrameLayout
    android:id="@+id/camera_preview"
    android:layout_width="fill_parent"
    android:layout_height="fill_parent"
    android:layout_weight="1"
    />

  <Button
    android:id="@+id/button_capture"
    android:text="Capture"
    android:layout_width="wrap_content"
    android:layout_height="wrap_content"
    android:layout_gravity="center"
    />
</LinearLayout>
```

在大多数设备上，相机的默认物理方向为横屏：

```kotlin
<activity android:name=".CameraActivity"
          android:label="@string/app_name"
          android:screenOrientation="landscape"/>
```

接下来的代码展示了**如何部署展示你的相机界面**：

```kotlin
class CameraActivity : Activity() {

    private var mCamera: Camera? = null
    private var mPreview: CameraPreview? = null

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContentView(R.layout.activity_main)

        // 该函数代码，请参考上面的【访问相机】小节
        mCamera = getCameraInstance()

        mPreview = mCamera?.let {
            // Create our Preview view
            CameraPreview(this, it)
        }

        // Set the Preview view as the content of our activity.
        mPreview?.also {
            val preview: FrameLayout = findViewById(R.id.camera_preview)
            preview.addView(it)
        }
    }
}
```

### 6.拍照

实现拍照，请使用`Camera.takePicture()`方法。 该方法采用三个从摄像机接收数据的参数, 要以`JPEG`格式接收数据，必须实现`Camera.PictureCallback`接口以接收图像数据并将其写入文件。以下代码显示了`Camera.PictureCallback`接口的基本实现，用于保存从相机接收的图像:

```kotlin
private val mPicture = Camera.PictureCallback { data, _ ->
    val pictureFile: File = getOutputMediaFile(MEDIA_TYPE_IMAGE) ?: run {
        Log.d(TAG, ("Error creating media file, check storage permissions"))
        return@PictureCallback
    }

    try {
        val fos = FileOutputStream(pictureFile)
        fos.write(data)
        fos.close()
    } catch (e: FileNotFoundException) {
        Log.d(TAG, "File not found: ${e.message}")
    } catch (e: IOException) {
        Log.d(TAG, "Error accessing file: ${e.message}")
    }
}
```

然后将这个`Callback`在合理的时机使用：

```kotlin
val captureButton: Button = findViewById(R.id.button_capture)
captureButton.setOnClickListener {
    // get an image from the camera
    mCamera?.takePicture(null, null, mPicture)
}
```

### 7.录制视频

> 录制视频的实现相对复杂，而官方文档提供的三言两语略显单薄，因此该内容不放在本小节中，我将在之后的一篇文章中进行阐述。

### 8.释放Camera资源

```kotlin
class CameraActivity : Activity() {
    private var mCamera: Camera?
    private var mPreview: SurfaceView?
    private var mMediaRecorder: MediaRecorder?

    override fun onPause() {
        super.onPause()
        releaseMediaRecorder() // if you are using MediaRecorder, release it first
        releaseCamera() // release the camera immediately on pause event
    }

    private fun releaseMediaRecorder() {
        mMediaRecorder?.reset() // clear recorder configuration
        mMediaRecorder?.release() // release the recorder object
        mMediaRecorder = null
        mCamera?.lock() // lock camera for later use
    }

    private fun releaseCamera() {
        mCamera?.release() // release the camera for other applications
        mCamera = null
    }
}
```

> 上述三篇文章主要来自Google官方的Camera文档，令人不解的是，官方的几篇文档互相之间链接不上，很多地方还有令人看不懂的参数，因此仅作为系统入门文章勉强适合，接下来的文章将会从不同角度阐述Camera使用开发过程中的一些疑点难点。
