# 卡顿优化

## 什么是卡顿

- 对于用户来说就是滑动、翻页时的不流畅。

- 在技术上来讲，就是掉帧了。正常的 iPhone 是大约 60FPS

## 为什么会产生卡顿

### 显示原理

- HSync: 水平同步信号（horizonal synchronization）,当扫描到新的一行
- VSync: 垂直同步信号（vertical synchronization）,当扫描到新的一帧

- 流程: CPU 计算好显示内容提交到 GPU，GPU 渲染完成后将渲染结果放入帧缓冲区，随后视频控制器会按照 VSync 信号逐行读取帧缓冲区的数据，经过可能的数模转换传递给显示器显示。

### 卡顿原因

在 VSync 信号到来后，系统图形服务会通过 CADisplayLink 等机制通知 App，App 主线程开始在 CPU 中计算显示内容，比如视图的创建、布局计算、图片解码、文本绘制等。随后 CPU 会将计算好的内容提交到 GPU 去，由 GPU 进行变换、合成、渲染。随后 GPU 会把渲染结果提交到帧缓冲区去，等待下一次 VSync 信号到来时显示到屏幕上。由于垂直同步的机制，如果在一个 VSync 时间内，CPU 或者 GPU 没有完成内容提交，则那一帧就会被丢弃，等待下一次机会再显示，而这时显示屏会保留之前的内容不变。这就是界面卡顿的原因。

### CPU 资源消耗原因

CPU 需要做的东西较多，比如对象的管理、布局的计算、文本的绘制、图片的解码等。

- 创建对象
  - 创建对象会分配内存、调整属性。尽量使用轻量的对象。 比如 CALayer 替代 UIView
  - 不涉及 UI 操作，尽量放到后台线程
  - 懒加载
  - 对象复用

- 对象调整
  - CALayer 内部没有属性，调用属性方法时，内部通过运行时的 resolveInstanceMethod 添加临时方法，并把对应属性值保存在内部的 Dictionary 中，同时通知 delegate、创建动画等。
  - 尽量避免调整视图层次、添加和移除视图。
  
- 对象销毁
    - 单一销毁消耗不多，对于容器类持有大量对象时，会比较明显
    - 移动到后台线程销毁

- 计算布局
    - 调整 UIView frame / bounds / center 等属性都是映射到 CALayer，很消耗性能
    - 尽量一次设置完成

- AutoLayout
    - 可以提高开发效率
    - 复杂视图严重影响性能

- 文本计算
    - 如果界面包含大量文本，文本的宽高计算会占用资源且无法避免。
    - 可以放到后台线程处理。

- 文本渲染
    - 底层都是 CoreText 排版、绘制为 Bitmap 显示的。常见的文本控件（UILabel、UITextView等），排版和绘制都是在主线程进行。大量文本时，CPU压力非常大。
    - 通过自定义文本控件解决， 用 TextKit 或者 CoreText 对文本异步绘制。

- 图片解码
    - UIImage 或 CGImageSource 等方法创建图片时，图片数据不会立即解码，只会设置到 CALayer.contents 中，在 CALayer 提交到 GPU 前， CGImage 中的数据才会解码。
    - 发生在主线程、且不可避免
    - 后台线程先把图片绘制到 CGBitmapContext 中，然后从 Bitmap 直接创建图片
    - 解码的原因：在磁盘里读取的是 PNG、JPG 等格式的图片，是经过编码压缩的，而显示在屏幕的数据需要是原始的像素数据。这个转化就是解码的过程。

- 图片的绘制
    - [UILabel drawRect:] [CALayer display] 等方法使用 CoreGraphic 方法进行图片绘制。
    - CG 线程安全，可以放到后台线程绘制。

### GPU 资源消耗原因

GPU 能做的东西比较单一： 接收提交的纹理（Textture）和定点描述，应用变换（transform）、混合（Composing）和渲染(Render)。

- 纹理的渲染
    - 所有的Bitmap，最终都是由内存提交到显存，绑定为 GPU Texture。
    - 短时间显示大量图片时，会 GPU 卡顿
    - 较少短时间大量图片使用
    - 尽量将多张图片合成一张显示
  
- 视图的混合
    - 多个视图重叠一起显示时，GPU 会先混合。
    - 视图结构层级不宜太复杂，较少视图数量和层次。
    - 标明 opaque，避免无用的 Alpha 通道合成。

- 图形的生成
    - CALayer 的 border、圆角、阴影、遮罩（mask），会触发离屏渲染(offscreen rendering)。

## 如何优化

- 列表
    - 缓存高度
    - 图片、文本异步处理
    - 有大量图片的界面快速滑动时，在低端机器暂停图片的加载

- 首次进入复杂页面的加载速度
    - 减少视图数量和层次
    - 预处理，提前处理一些复杂操作
    - 懒加载，延迟操作非必须操作

- 频繁调用的地方
    - ScrollView DidScroll 尽量减少耗时操作
    - 耗时操作缓存结果

- 优化业务流程
    - 例如较少数据库的使用
    - 算法优化
    - 非紧急事务错峰操作（比如网络返回时需大量数据处理，可以延时打点）

- 合理管理线程
    - GCD 管理的线程池，无法直接操作。如果并行队列有任务阻塞或者太多的串行队列同时执行，会创建大量的线程。
    - 每个线程需要分配内存和内核内存,主线程占1M,子线程占用512KB,1KB的内核内存
    - 每个线程大概需要 90 ms 的创建时间
    - 过多线程会导致大量的上下文切换
    - 合理的线程分配，最终目的就是保证主线程尽量少的处理非UI操作，同时控制整个App的子线程数量在合理的范围内。

- 正确使用 API
    - 了解 imageNamed: 与 imageWithContentsOfFile:的差异
    - 缓存 NSDateFormatter 的结果。
    - 不能使用 NSLog()
    - 当试图获取磁盘中一个文件的属性信息时，使用 [NSFileManager attributesOfItemAtPath:error:] 会浪费大量时间读取可能根本不需要的附加属性。这时可以使用 stat 代替 NSFileManager，直接获取文件属性

- 图片
    - 减少离屏渲染
        - 只裁剪bg，不 maskToBounds
        - 圆角使用蒙层
        - ShadowPath替代shadowOffset
    - 像素对齐：图片不要使用小数点, 否则需要进行插值计算
    - 图片格式: 苹果的GPU只解析32bit的颜色格式。其他格式要先经过CPU处理
    - 后台读取、解码、缩放
    - 图片缓存：内存缓存，加快读取速度

- 文本
    - 异步绘制

- IO
    - 数据库异步读写
    - 文件异步读写 

## 如何监控

Instruments Time Profiler

- 检查占用CPU较高的方法

Instruments Core Animation

- 检查离屏渲染、像素对齐、图片缩放、格式是否支持

自制监控工具

- 检查 CPU、Mem、FPS等
- 主线程超过 100ms 没有响应时，打印堆栈
- 更多查看 [性能监控](performance_monitor.md)

代码业务逻辑
    - 高频调用方法逻辑合理性
    - CPU占用高的方法算法逻辑优化
    - 打印日志，查看流程是否合理

## 总结

- 尽量把可以异步执行的操作，都放到后台线程处理。
- 性能问题不可避免，需要提升团队整体的意识
- 需在 CI 中加入性能测试，一切以数据说话

## 参考文档

[iOS 保持界面流畅的技巧 - ibireme](https://blog.ibireme.com/2015/11/12/smooth_user_interfaces_for_ios/)

[微信读书性能优化](http://wereadteam.github.io/2016/05/03/WeRead-Performance/)

[微信iOS卡顿监控系统](https://mp.weixin.qq.com/s/M6r7NIk-s8Q-TOaHzXFNAw)

[谈谈 iOS 中图片的解压缩 - 雷纯锋](http://blog.leichunfeng.com/blog/2017/02/20/talking-about-the-decompression-of-the-image-in-ios/)