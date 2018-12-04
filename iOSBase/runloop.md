##RunLoop

###什么是RunLoop

一般来讲，一个线程一次只能执行一个任务，执行完成后线程就会退出。如果我们需要一个机制，让线程能随时处理事件但并不退出

RunLoop是Event Loop

一直循环，执行完任务就休眠，有事件就唤醒执行时间，执行完继续休眠。

###RunLoop有什么优势
- 线程一直运行并接收事件
- 节省CPU时间，操作系统按照时间片调用，睡眠后CPU不需要分配时间

###RunLoop对象
NSRunLoop 和 CFRunLoopRef

CoreFoundation 是开源的。

调用到了系统的一些方法
`GCD` `mach kernal` `block` `pthread` ...


### RunLoop组成
#### Mode
- RunLoop必须在一个且只能在一个mode中run
- 更换mode时，需停止当前loop，然后重新run loop
- mode是iOS滑动顺畅的关键
- 可自定义mode（基本不会用）

常见mode：

- NSDefaultRunLoopMode
- UITrackingRunLoopMode
- NSRunLoopCommonModes

Common Modes:

一个mode可以将自己标记为“Common”（将自己的ModeName添加到RunLoop的“commonModes”数组中）。mode items 添加事指定commonModes的话，会被添加到RunLoop下的“commonModeItems”中，里面的items会同步到所有标记为common的mode里。

#### Mode items

#### Observer
- entry - 即将进入loop
- beforeTimers - 即将处理 timer
- beforeSources - 即将处理 source
- beforeWaiting - 即将进入休眠
- afterWaiting - 休眠状态中唤醒
- exit - 即将退出loop

#### Timer
CFRunLoopTimer

- NSTimer
- performSelector:delay:
- CADisplayLink

#### Source0 
CFRunLoopSource

- 处理App内部事件
- UIEvent
- CFSocket

####Source1
- RunLoop和内核管理
- Mach port 驱动

### RunLoop的唤醒和挂起
为了实现消息的发送和接收，`mach_msg()` 函数实际上是调用了一个 Mach 陷阱 (trap)，即函数`mach_msg_trap()`，陷阱这个概念在 Mach 中等同于系统调用。当你在用户态调用 `mach_msg_trap()` 时会触发陷阱机制，切换到内核态；内核态中内核实现的 `mach_msg()` 函数会完成实际的工作

###RunLoop的一些应用

#### AutoreleasePool
- Entry时 `_objc_autoreleasePoolPush()`
- BeforeWaiting 时 `_objc_autoreleasePoolPop()` &  `_objc_autoreleasePoolPush()`
- Exit时 `_objc_autoreleasePoolPop()`

####界面更新
苹果注册了一个 Observer 监听 BeforeWaiting(即将进入休眠) 和 Exit (即将退出Loop) 事件，回调去执行一个很长的函数：
_ZN2CA11Transaction17observer_callbackEP19__CFRunLoopObservermPv()。这个函数里会遍历所有待处理的 UIView/CAlayer 以执行实际的绘制和调整，并更新 UI 界面。

```objc
_ZN2CA11Transaction17observer_callbackEP19__CFRunLoopObservermPv()
    QuartzCore:CA::Transaction::observer_callback:
        CA::Transaction::commit();
            CA::Context::commit_transaction();
                CA::Layer::layout_and_display_if_needed();
                    CA::Layer::layout_if_needed();
                        [CALayer layoutSublayers];
                            [UIView layoutSubviews];
                    CA::Layer::display_if_needed();
                        [CALayer display];
                            [UIView drawRect];
```

####事件响应
当一个硬件事件(触摸/锁屏/摇晃等)发生后，首先由 IOKit.framework 生成一个 IOHIDEvent 事件并由 SpringBoard 接收。然后通过Mach port分发到需要的进程，然后注册的Source1就会触发回调。然后通过_UIApplicationHandleEventQueue() 进行事件分发。

#### NSTimer
Timer是RunLoop的一个事件源，如果没有RunLoop不会生效，如果RunLoop当前的Mode中不包含改Timer也不会生效。

#### UIEvent


####性能优化
滑动时不加载图片：

```objc
UIImage *downLoadImage = ...;  
[self.avatarImageView performSelector:@selector(setImage:)  
                        withObject:downloadImage  
                        afterDelay:0  
                        inModes:@[NSDefaultRunLoopMode]];
```


#### 线程保活
```objc
+ (void)networkRequestThreadEntryPoint:(id)__unused object {
    @autoreleasepool {
        [[NSThread currentThread] setName:@"AFNetworking"];
        NSRunLoop *runLoop = [NSRunLoop currentRunLoop];
        [runLoop addPort:[NSMachPort port] forMode:NSDefaultRunLoopMode];
        [runLoop run];
    }
}
 
+ (NSThread *)networkRequestThread {
    static NSThread *_networkRequestThread = nil;
    static dispatch_once_t oncePredicate;
    dispatch_once(&oncePredicate, ^{
        _networkRequestThread = [[NSThread alloc] initWithTarget:self selector:@selector(networkRequestThreadEntryPoint:) object:nil];
        [_networkRequestThread start];
    });
    return _networkRequestThread;
}
```

