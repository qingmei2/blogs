> Camera系列文章首发于 [我的慕课网](https://www.imooc.com/u/6936865)，欢迎关注。

## 概述
 **Camera** 可能是接下来个人想深入学习的课题，准备新起一个系列，从个人的角度总结阐述自己对于 **Android Camera** 的研究过程，希望也能够对其他想学习 **Camera** 的同学一些帮助。

> 本小节内容为 **Android Camera 官方文档** 的精要翻译，原文请参考：
> * [Android Camera: Take photos](https://developer.android.com/training/camera/photobasics)
> * [Android Camera: Record videos](https://developer.android.com/training/camera/videobasics)

## 一、拍照

本课程将阐述如何通过委托Android设备上的其他相机应用程序进行拍照 （如果您更愿意构建自己的相机功能，请参阅 [控制相机](https://developer.android.com/training/camera/cameradirect) ）。

### 请求相机功能

如果您的应用程序的基本功能涉及到 **拍照**，请将其在Google Play上的可见性限制为具有相机的设备。 以声明您的应用程序依赖于摄像头，请在清单文件中放置`<uses-feature>`标记。

```xml
<manifest ... >
    <uses-feature android:name="android.hardware.camera"
                  android:required="true" />
    ...
</manifest>
```

### 使用其他相机APP拍照

你可以通过Android的`Intent`将拍照行为委托给其他的拍照应用， 此过程涉及三个部分：`Intent`本身，调用并启动外部`Activity`，以及在`Activity`中处理回调的数据。


下面是调用启动拍照应用的函数代码：

```kotlin
val REQUEST_IMAGE_CAPTURE = 1

private fun dispatchTakePictureIntent() {
    Intent(MediaStore.ACTION_IMAGE_CAPTURE).also { takePictureIntent ->
        takePictureIntent.resolveActivity(packageManager)?.also {
            startActivityForResult(takePictureIntent, REQUEST_IMAGE_CAPTURE)
        }
    }
}
```

请注意，调用`startActivityForResult `函数之前，请先通过调用`resolveActivity `函数以保证`startActivityForResult `函数中的`Intent`能够被正确的处理，否则将会导致应用的崩溃。

### 获取缩略图

Android Camera应用程序将返回的`Intent`中的照片通过`onActivityResult()`返回，作为附加内容中的一个小位图，位于关键字`data`下。 以下代码检索此结果并将其显示在`ImageView`中：

```kotlin
override fun onActivityResult(requestCode: Int, resultCode: Int, data: Intent) {
    if (requestCode == REQUEST_IMAGE_CAPTURE && resultCode == RESULT_OK) {
        val imageBitmap = data.extras.get("data") as Bitmap
        mImageView.setImageBitmap(imageBitmap)
    }
}
```
### 完整保存拍照结果

通常，用户使用设备摄像头拍摄的任何照片都应保存在公共外部存储设备中，以便所有应用都可以访问。 共享照片的正确目录由`getExternalStoragePublicDirectory()`提供，带有`DIRECTORY_PICTURES`参数。 由于此方法提供的目录在所有应用程序之间共享，因此读取和写入该目录分别需要`READ_EXTERNAL_STORAGE`和`WRITE_EXTERNAL_STORAGE`权限。 写权限隐式允许读取，因此如果您需要写入外部存储，那么您只需要请求一个权限：

```xml
<manifest ...>
    <uses-permission android:name="android.permission.WRITE_EXTERNAL_STORAGE" />
    ...
</manifest>
```

但是，如果您希望照片仅保留为应用程序的私密照片，则可以使用`getExternalFilesDir()`提供的目录。 在Android 4.3及更低版本中，写入此目录还需要`WRITE_EXTERNAL_STORAGE`权限。 从Android 4.4开始，不再需要该权限，因为其他应用程序无法访问该目录，因此您可以通过添加`maxSdkVersion`来声明仅在较低版本的Android上请求权限：

```xml
<manifest ...>
    <uses-permission android:name="android.permission.WRITE_EXTERNAL_STORAGE"
                     android:maxSdkVersion="18" />
    ...
</manifest>
```

> 注意：当用户卸载应用程序时，将删除由`getExternalFilesDir()`或`getFilesDir()`提供的目录中保存的文件。

确定文件的目录后，需要创建一个防冲突的文件名。 您可能还希望将路径保存在成员变量中以供以后使用。 以下解决方案是通过时间戳为新照片返回唯一文件名的示例：

```kotlin
var mCurrentPhotoPath: String

@Throws(IOException::class)
private fun createImageFile(): File {
    // Create an image file name
    val timeStamp: String = SimpleDateFormat("yyyyMMdd_HHmmss").format(Date())
    val storageDir: File = getExternalFilesDir(Environment.DIRECTORY_PICTURES)
    return File.createTempFile(
            "JPEG_${timeStamp}_", /* prefix */
            ".jpg", /* suffix */
            storageDir /* directory */
    ).apply {
        // Save a file: path for use with ACTION_VIEW intents
        mCurrentPhotoPath = absolutePath
    }
}
```

使用此方法可以为照片创建文件，您现在可以像这样创建和调用Intent：

```kotlin
val REQUEST_TAKE_PHOTO = 1

private fun dispatchTakePictureIntent() {
    Intent(MediaStore.ACTION_IMAGE_CAPTURE).also { takePictureIntent ->
        // 保证intent可正确的跳转
        takePictureIntent.resolveActivity(packageManager)?.also {
            // 创建保存照片的文件路径
            val photoFile: File? = try {
                createImageFile()
            } catch (ex: IOException) {
                // 处理异常
                ...
                null
            }
            // 仅在成功创建文件时继续
            photoFile?.also {
                val photoURI: Uri = FileProvider.getUriForFile(
                        this,
                        "com.example.android.fileprovider",
                        it
                )
                takePictureIntent.putExtra(MediaStore.EXTRA_OUTPUT, photoURI)
                startActivityForResult(takePictureIntent, REQUEST_TAKE_PHOTO)
            }
        }
    }
}
```

> 注意：我们使用 `getUriForFile（Context，String，File）`返回`content：// URI`。 对于针对Android 7.0（API级别24）及更高版本的更新应用，在包边界上传递`file:// URI`会导致`FileUriExposedException`。 因此，我们现在提供一种使用 [FileProvider](https://developer.android.com/reference/android/support/v4/content/FileProvider) 存储图像的更通用的方法。

现在，您需要配置 [FileProvider](https://developer.android.com/reference/android/support/v4/content/FileProvider) 。 在您的应用清单中，向您的应用添加对应的`Provider`：

```xml
<application>
   ...
   <provider
        android:name="android.support.v4.content.FileProvider"
        android:authorities="com.example.android.fileprovider"
        android:exported="false"
        android:grantUriPermissions="true">
        <meta-data
            android:name="android.support.FILE_PROVIDER_PATHS"
            android:resource="@xml/file_paths"></meta-data>
    </provider>
    ...
</application>
```

确保将`authority`字符串与`getUriForFile（Context，String，File）`的第二个参数匹配。 在APP的 `meta-data`中，您可以看到APP期望在资源文件`res/xml/file_paths.xml`中配置符合条件的 path。 以下是此示例所需的代码内容：

```xml
<?xml version="1.0" encoding="utf-8"?>
<paths xmlns:android="http://schemas.android.com/apk/res/android">
    <external-path name="my_images" path="Android/data/com.example.package.name/files/Pictures" />
</paths>
```

### 将照片加入相册

当您通过意图创建照片时，您应该知道其所在位置，因为您首先要说明将图像保存在何处。 对于其他所有人来说，使照片可以访问的最简单方法可能是从系统的相册访问它：

 ```kotlin
// 将图片保存到相册
private fun galleryAddPic() {
    Intent(Intent.ACTION_MEDIA_SCANNER_SCAN_FILE).also { mediaScanIntent ->
        val f = File(mCurrentPhotoPath)
        mediaScanIntent.data = Uri.fromFile(f)
        sendBroadcast(mediaScanIntent)
    }
}
```

### 解码缩放图片

管理多个全尺寸的图片可能会因内存有限而变得棘手。 如果在显示几个图片后发现应用程序内存不足，则可以通过将图片压缩减少动态堆的使用量。 以下示例方法演示了此技术：

```kotlin
private fun setPic() {
    // Get the dimensions of the View
    val targetW: Int = mImageView.width
    val targetH: Int = mImageView.height

    val bmOptions = BitmapFactory.Options().apply {
        // Get the dimensions of the bitmap
        inJustDecodeBounds = true
        BitmapFactory.decodeFile(mCurrentPhotoPath, this)
        val photoW: Int = outWidth
        val photoH: Int = outHeight

        // Determine how much to scale down the image
        val scaleFactor: Int = Math.min(photoW / targetW, photoH / targetH)

        // Decode the image file into a Bitmap sized to fill the View
        inJustDecodeBounds = false
        inSampleSize = scaleFactor
        inPurgeable = true
    }
    BitmapFactory.decodeFile(mCurrentPhotoPath, bmOptions)?.also { bitmap ->
        mImageView.setImageBitmap(bitmap)
    }
}
```

## 二、视频录制

### 请求相机功能

如果您的应用程序的基本功能涉及到 **拍照**，请将其在Google Play上的可见性限制为具有相机的设备。 以声明您的应用程序依赖于摄像头，请在清单文件中放置`<uses-feature>`标记。

```xml
<manifest ... >
    <uses-feature android:name="android.hardware.camera"
                  android:required="true" />
    ...
</manifest>
```

### 使用相机应用录制视频

```kotlin
const val REQUEST_VIDEO_CAPTURE = 1

private fun dispatchTakeVideoIntent() {
    Intent(MediaStore.ACTION_VIDEO_CAPTURE).also { takeVideoIntent ->
        takeVideoIntent.resolveActivity(packageManager)?.also {
            startActivityForResult(takeVideoIntent, REQUEST_VIDEO_CAPTURE)
        }
    }
}
```

### 观看视频

```kotlin
override fun onActivityResult(requestCode: Int, resultCode: Int, intent: Intent) {
    if (requestCode == REQUEST_VIDEO_CAPTURE && resultCode == RESULT_OK) {
        val videoUri: Uri = intent.data
        mVideoView.setVideoURI(videoUri)
    }
}
```
