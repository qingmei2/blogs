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

比如，如果编码是 `PCM-16`，那么这个比例将是 `16 bit`，所以我们可以在 `2` 的 `16` 次幂的数字范围内表示一个幅度，即 `65536` 个不同的幅度值。