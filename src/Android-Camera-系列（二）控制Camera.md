> Camera系列文章首发于 [我的慕课网](https://www.imooc.com/u/6936865)，欢迎关注。

## 概述
 **Camera** 可能是接下来个人想深入学习的课题，准备新起一个系列，从个人的角度总结阐述自己对于 **Android Camera** 的研究过程，希望也能够对其他想学习 **Camera** 的同学一些帮助。

> 本小节内容为 **Android Camera 官方文档** 的精要翻译，原文请参考：
> * [Android Camera: Control the camera](https://developer.android.com/training/camera/cameradirect)

## 一、打开相机对象

获取`Camera`对象的实例是自定义相机的第一步。 作为Android的 **自定义相机应用**，更推荐是启动一个单独的线程并在`onCreate()`生命周期的回调打开`Camera`。 打开摄像机的操作延迟到`onResume()`方法中处理是一个不错的注意，因为`Camera`的初始化可能需要一段时间，而在UI线程这样做则有可能让应用变得卡顿。 

如果摄像头已被其他应用程序使用，则调用`Camera.open()`会抛出异常，因此我们将其包装在`try{}`中。

```kotlin
private fun safeCameraOpen(id: Int): Boolean {
    return try {
        releaseCameraAndPreview()
        mCamera = Camera.open(id)
        true
    } catch (e: Exception) {
        Log.e(getString(R.string.app_name), "failed to open Camera")
        e.printStackTrace()
        false
    }
}

private fun releaseCameraAndPreview() {
    mPreview?.setCamera(null)
    mCamera?.also { camera ->
        camera.release()
        mCamera = null
    }
}
```

## 二、创建相机预览

### 预览类
要开始显示相机的预览，您需要预览类。 预览需要实现`android.view.SurfaceHolder.Callback`接口，该接口用于将图像数据从相机硬件传递到应用程序。

```kotlin
class Preview(
        context: Context,
        val mSurfaceView: SurfaceView = SurfaceView(context)
) : ViewGroup(context), SurfaceHolder.Callback {

    var mHolder: SurfaceHolder = mSurfaceView.holder.apply {
        addCallback(this@Preview)
        setType(SurfaceHolder.SURFACE_TYPE_PUSH_BUFFERS)
    }
    ...
}
```

必须先将预览类传递给`Camera`对象，然后才能启动实时图像预览，如下一节所示。

### 配置并开始预览

`Camera`的配置和预览等相关操作都需要**严格地控制顺序**，`Camera`对象首当其冲， 在下面的代码片段中，`Camera`的初始化过程被封装，以便每当用户做某事来更改`Camera`都会调用`setCamera()`方法，即调用`Camera.startPreview()`。 还必须在预览类的`surfaceChanged()`回调方法中重新启动预览。

```kotlin
fun setCamera(camera: Camera?) {
    if (mCamera == camera) {
        return
    }

    stopPreviewAndFreeCamera()

    mCamera = camera

    mCamera?.apply {
        mSupportedPreviewSizes = parameters.supportedPreviewSizes
        requestLayout()

        try {
            setPreviewDisplay(mHolder)
        } catch (e: IOException) {
            e.printStackTrace()
        }

        // Important: Call startPreview() to start updating the preview
        // surface. Preview must be started before you can take a picture.
        startPreview()
    }
}
```

## 三、修改Camera的配置

相机设置会改变相机拍摄照片的方式，从**缩放级别**到**曝光补偿**。 下述代码示例仅展示了如何更改预览大小：

```kotlin
override fun surfaceChanged(holder: SurfaceHolder, format: Int, w: Int, h: Int) {
    mCamera?.apply {
        // Now that the size is known, set up the camera parameters and begin
        // the preview.
        parameters?.also { params ->
            params.setPreviewSize(mPreviewSize.width, mPreviewSize.height)
            requestLayout()
            parameters = params
        }

        // Important: Call startPreview() to start updating the preview surface.
        // Preview must be started before you can take a picture.
        startPreview()
    }
}
```

## 四、设置预览方向

大多数相机应用程序将显示器锁定为**横向模式**，因为这是**相机传感器的物理方向**。 此设置不会阻止您拍摄纵向模式照片，因为设备的方向记录在**EXIF Header**中。 [setCameraDisplayOrientation()](https://developer.android.com/reference/android/hardware/Camera.html#setDisplayOrientation(int)) 方法允许您更改预览的显示方式，而不会影响图片的记录方式。

```kotlin
// kotlin版本
@TargetApi(Build.VERSION_CODES.ICE_CREAM_SANDWICH)
fun setCameraDisplayOrientation(activity: Activity,
                                    cameraId: Int = 0,
                                    camera: Camera) {
       val info = Camera.CameraInfo()
       Camera.getCameraInfo(cameraId, info)
       val rotation = activity.windowManager.defaultDisplay.rotation
       var degrees = 0
       when (rotation) {
           Surface.ROTATION_0 -> degrees = 0
           Surface.ROTATION_90 -> degrees = 90
           Surface.ROTATION_180 -> degrees = 180
           Surface.ROTATION_270 -> degrees = 270
       }
       var result: Int
       if (info.facing == Camera.CameraInfo.CAMERA_FACING_FRONT) {
           result = (info.orientation + degrees) % 360
           result = (360 - result) % 360  // compensate the mirror
       } else {  // back-facing
           result = (info.orientation - degrees + 360) % 360
       }
       camera.setDisplayOrientation(result)
}
```
## 五、拍照

预览开始展示后，可以使用`Camera.takePicture()`方法拍摄照片。 您可以创建`Camera.PictureCallback`和`Camera.ShutterCallback`对象，并将它们传递给`Camera.takePicture()`。

如果要连续抓取图像，可以创建一个实现`onPreviewFrame()`的`Camera.PreviewCallback`。 对于介于两者之间的内容，您可以仅捕获选定的预览帧，或设置延迟操作以调用`takePicture()`。

## 六、重新启动预览

拍摄照片后，必须重新开始预览，然后用户才能拍摄另一张照片。 下述代码展示了如何通过重载快门按钮完成重启：

```kotlin
fun onClick(v: View) {
    mPreviewState = if (mPreviewState == K_STATE_FROZEN) {
        mCamera?.startPreview()
        K_STATE_PREVIEW
    } else {
        mCamera?.takePicture(null, rawCallback, null)
        K_STATE_BUSY
    }
    shutterBtnConfig()
}
```

## 七、停止预览并释放相机

当`Camera`功能使用完毕后，释放对应的资源保证相机能被其它应用所再次使用，至于释放时机，在预览View的`surfaceDestroyed()`处理是个不错的选择。

```kotlin
override fun surfaceDestroyed(holder: SurfaceHolder) {
    // Surface will be destroyed when we return, so stop the preview.
    // Call stopPreview() to stop updating the preview surface.
    mCamera?.stopPreview()
}

/**
 * When this function returns, mCamera will be null.
 */
private fun stopPreviewAndFreeCamera() {
    mCamera?.apply {
        // Call stopPreview() to stop updating the preview surface.
        stopPreview()

        // Important: Call release() to release the camera for use by other
        // applications. Applications should release the camera immediately
        // during onPause() and re-open() it during onResume()).
        release()

        mCamera = null
    }
}
```

> **笔者注**：官方文档关于《控制Camera》小节的课程文档，归纳得实在是无力吐槽，关于`Camera`各项参数的配置方式请参考笔者接下来的文章，本小节的作用请将其视为Camera相关知识的**系统化了解**。
