# Optimization
iOS性能优化点及优化技巧
## 界面流畅
### 显示原理
* iOS 使用双缓存、垂直同步机制。
双缓存机制是为了最大程度上利用GPU，提升帧缓冲区的读取和刷新效率。垂直同步信号是为了解决画面撕裂现象。
* 卡顿产生
![display](/DIsplayProcess.png)
在 VSync 信号到来后，系统图形服务会通过 CADisplayLink 等机制通知 App，App 主线程开始在 CPU 中计算显示内容，比如视图的创建、布局计算、图片解码、文本绘制等。随后 CPU 会将计算好的内容提交到 GPU 去，由 GPU 进行变换、合成、渲染。随后 GPU 会把渲染结果提交到帧缓冲区去，等待下一次 VSync 信号到来时显示到屏幕上。由于垂直同步的机制，如果在一个 VSync 时间内，CPU 或者 GPU 没有完成内容提交，则那一帧就会被丢弃，等待下一次机会再显示，而这时显示屏会保留之前的内容不变。这就是界面卡顿的原因。

从上面的图中可以看到，CPU 和 GPU 不论哪个阻碍了显示流程，都会造成掉帧现象。所以开发时，也需要分别对 CPU 和 GPU 压力进行评估和优化。

### 离屏渲染
### layout
### 文本绘制
### 复杂界面
### 图片解码
## 数据库优化(Coredata)
## 网络优化(Https)
