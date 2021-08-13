# [译] Android 可视化器的自定义实现

> 原文：[An alternative Android Visualizer](https://www.egeniq.com/blog/alternative-android-visualizer)  
> 作者：[Dániel Zolnai](https://www.egeniq.com/blog/alternative-android-visualizer)  
> 译者：[却把清梅嗅](https://github.com/qingmei2)  

听音乐时，有时你会看到那些视觉上令人愉悦的悦动条，它们音量越大跳得越高。通常，左边的条形对应的频率较低（低音），而右边的条形对应较高的频率（高音）：

![](image1.gif)

这些悦动条通常被称为 **视觉均衡器** 或 **可视化器**，若想在 `Android` 应用中展示类似的可视化效果，你可以使用 `Android` 原生的 `Visualizer` 类，它是`Android`框架中的一部分，且能够附加到你的 `AudioTrack`。

它是切实有效的，但有一个重要的缺陷：它需要申请 **麦克风权限** ，而从官方文档上来看，这是有确切考虑的：

> To protect privacy of certain audio data (e.g voice mail) the use of the visualizer requires the permission.
>
>为了保护某些音频数据（例如语音邮件）的隐私，使用 `Visualizer` 需要获取权限。

问题是，用户不会允许音乐 `APP` 申请使用他们的麦克风权限（这毫无疑问）。而当我翻遍了 `Android` 官方提供的`API`或者其他三方库，却找不到实现这样 **可视化器** 效果的替代方案。

因此我考虑自己造轮子，第一个问题是，我需要思考如何将正在播放的音乐，转换成每个跳跃条对应的高度。

## 可视化器的工作原理

首先，让我们从输入开始。当数字化音频时，我们通常会对信号幅度进行非常频繁的采样，这称为**脉冲编码调制** (PCM)。振幅随之被量化，我们将其表示到我们自己的 **数字标度** 上。

举个例子，如果编码是 `PCM-16`，这个比例将是 `16 bit`，我们可在 `2` 的 `16` 次幂的数字范围内表示一个幅度，即 `65536` 个不同的幅度值。

如果您在多个 `channel` 上采样（如立体声，分别录制左右声道），这些幅度会相互跟随，因此首先是 `channel 0` 的幅度，然后是 `channel 1` 的幅度，然后是 `channel 0`，依此类推。 一旦我们获得了这些幅度值作为原始数据，我们就可以继续下一步。 为此，我们需要了解声音实际上是什么：

> 我们听到的声音是物体振动的结果。例如人的声带、吉他的金属弦和木琴身。一般情况下，若不受特定声音振动的影响，空气分子会随机移动。  
>  
> 选自[《 Digital Sound and Music 》](http://digitalsoundandmusic.com/2-1-1-sound-waves-sine-waves-and-harmonic-motion/)

当敲击音叉时，它会以非常特定的 `440 次/秒 (Hz)` 振动，这种振动将通过空气传播到耳膜，在那里以相同的频率共振，大脑会将其解释为音符A。

在 `PCM` 中，这可以表示为正弦波，每秒重复 `440` 次。这些波的高度不会改变音符，但它们代表振幅；通俗点说，就是当听到它时，你耳朵里的响度。

但是当听音乐时，通常不仅有正在听的`音符A`（虽然我希望这样），而且还有过多的乐器和声音，从而导致 `PCM` 图形对人眼没有意义。实际上它是不同频率和振幅的不同正弦波大量振动的组合。

即使是非常简单的 `PCM` 信号（例如方波）在解构为不同的正弦波时也非常复杂：

![](image2.gif)

> 方波解构为近似正弦和余弦波，参考自 [这里](https://visualizingmath.tumblr.com/post/63962473846/1ucasvb-the-fourier-transform-takes-an-input) 。

幸运的是，我们有算法来进行这种解构，我们称之为 **傅立叶变换** 。正如上文可视化器所展示的，它实际上是从正弦波和余弦波的组合中解构而出的。余弦基本上是一个 **延迟** 的正弦波，但是在这个算法中拥有它们非常有用，否则我们将无法为点 `0` 创建一个值，因为每个正弦波都是从 `0` 开始的，相乘仍然会得到 `0`。

执行 **傅里叶变换** 的算法之一是 **快速傅里叶变换 ( FFT )**。 在我们的 `PCM` 声音数据上运行此 `FFT` 算法时，我们将获得每个正弦波的幅度列表。这些波是声音的频率。在列表的开头，我们可以找到低频（低音），最后是高频（高音）。

这样，我们通过绘制一个这样的条形图，其高度由每个频率的幅度决定——我们得到了我们想要的可视化器。

## 技术细节

现在回到 `Android`。 首先，我们需要音频的 `PCM` 数据。 为此，我们可以将 `AudioProcessor` 配置给到我们的 `ExoPlayer` 实例，它会在转发之前接收每个音频字节。您还可以进行修改，例如更改 **幅度** 或 **过滤通道**，但不是现在。

```Kotlin
private val fftAudioProcessor = FFTAudioProcessor()

val renderersFactory = object : DefaultRenderersFactory(this) {
    override fun buildAudioProcessors(): Array<AudioProcessor> {
        val processors = super.buildAudioProcessors()
        return processors + fftAudioProcessor
    }
}
player = ExoPlayerFactory.newSimpleInstance(this, renderersFactory, DefaultTrackSelector())
```

在 `queueInput(inputBuffer: ByteBuffer)` 方法中，我们将收到捆在一起作为一帧的 `byte` 数据。

这些 `byte` 可能来自多个 `channel`，为此我取了所有 `channel` 的平均值，并且仅将其转发以进行处理。