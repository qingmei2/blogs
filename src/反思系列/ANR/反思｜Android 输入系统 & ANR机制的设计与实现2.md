# 反思｜Android 输入系统 & ANR机制的设计与实现

## 四、ANR机制的设计与实现


分发过程共使用到3个事件队列：

* mInBoundQueue：用于记录`InputReader`发送过来的输入事件；
* outBoundQueue：用于记录即将分发给目标应用窗口的输入事件；
* waitQueue：用于记录已分发给目标应用，且应用尚未处理完成的输入事件。
