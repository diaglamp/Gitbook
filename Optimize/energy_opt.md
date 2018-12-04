# 耗电优化

## 检测方法

#### 设置->电池 

能看一个应用的大概耗电（跟使用频率也有关系）

#### Xcode， Debug Navigator 

可以定性测量

- Low 
- High
- Very High

影响耗电的主要因素是

- Overhead
- CPU
- Network
- Location
- GPU

#### Instruments 的 Energy Log

可以得到电池损耗日志，但只是一个 level （分母为20），并没有什么卵用

#### IOKit 私有接口

```
    AdapterInfo = 16384;
    AtCriticalLevel = 0;
    BatteryInstalled = 1;
    CurrentCapacity = 1800; 当前的剩余电量
    ExternalChargeCapable = 1;
    ExternalConnected = 1;
    FullyCharged = 0;
    IOClass = AppleARMPMUCharger;
    IsCharging = 0;
    MaxCapacity = 1800; 最大电池容量
    Voltage = 4255;
```

## 影响因子

官方文档说：

- CPU
- Device wake
- Networking operations
- Graphics, animations, and video.
- Location
- Motion
- Bluetooth

WWDC 视频说：

- CPU （Processing）
- 网络（Networking）
- 定位（Location）
- 图像（Graphics）

## 优化

### Cost
 
- `Fixed cost` - 就算只做一丢丢操作，都会有一个固定的消耗，这个硬件维持工作所需要的能耗。
- `Dynamic cost` 指的是具体工作量带来的能耗

### 节省电量的基本原则

- Identify 想清楚你需要 app 在特定时刻需要完成哪些工作，如果是不必要的工作，考虑延后执行或者省去。
- Optimize 优化 app 的功能实现，尽可能以更有效率的方式去完成功能。
- Coalesce 合并操作。
- Reduce 减少做重复工作的频率。

### 具体的优化操作

- 减少网络请求频率
    - 减少重复
    - 降低超时时间
    - 使用 background session
    - 批量处理
    - 减少重试次数
  
- 定位、陀螺仪、蓝牙等，用完就关。

- 图像处理 
    - 在真的需要刷新时才刷新
    - 尽量少 blur
  
- CPU 
    - 明确任务，优化流程
    - 高效完成
    - 尽量少使用定时器
    - 定时器设置 tolerance

再者，需要区分场景

- 在前台工作时，正常使用
- 在后台工作时，需要关闭一些操作
- 在特殊的页面（比如直播、视频之类）需要特别优化

## 参考


[Energy Efficiency Guide for iOS Apps](https://developer.apple.com/library/archive/documentation/Performance/Conceptual/EnergyGuide-iOS/FundamentalConcepts.html#//apple_ref/doc/uid/TP40015243-CH4-SW1)

[Writing Energy Efficient Apps](https://developer.apple.com/videos/play/wwdc2017/238/)