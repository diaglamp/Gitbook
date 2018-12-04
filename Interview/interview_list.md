# iOS Interview List

## OBJC

### 基础

#### unsafe_unretained, weak, assign区别

#### 64bit下NSNumber的优化

#### Tagged Pointer



### 调试

#### 怎么分析Crash日志

#### 一般怎样调试野指针


### 内存管理

#### 用两种方式造成循环引用

#### 对ARC的理解

### 多线程

#### NSThread, NSOperation, GCD区别

#### GCD执行原理

#### 造一个死锁


### Block

#### Block的底层原理

### 网络

### runloop

#### runloop内部实现逻辑

#### Autorelease (其实不完全属于runloop 知识点)

### runtime

#### 内存布局

#### 如何设计KVO

#### runtime应用场景

#### runtime如何通过selector找到对应的IMP地址

### 系统流程

#### 编译过程

#### 启动流程

#### 渲染过程


### 实际开发

#### 内存警告处理

#### 后台挂起处理事务



## 性能优化

### 性能监控

[性能监控](performance_monitor.md)

### 编译优化

[编译优化](compile_opt.md)

### 包大小优化

[包大小优化](ipa_opt.md)

### 启动优化

[启动优化](launch_opt.md)

### 卡顿优化



### 耗电优化

### 卡顿率优化

### 崩溃率优化







## 第三方库
### SDWebImage

### AFNetworking

### YYKit

## 数据结构与算法

### 最短路径算法

### 排序算法

### 两个整数最大公约数

### 二叉树遍历

## 其他
### Git使用

### Cocoapods

### 举一个例子遇到的问题

### 如何理解组件化解耦

## 代码分析
以下代码会导致什么问题

```objc
@property (nonatomic, strong) NSString *target;
//....
dispatch_queue_t queue = dispatch_queue_create("parallel", DISPATCH_QUEUE_CONCURRENT);
for (int i = 0; i < 1000000 ; i++) {
    dispatch_async(queue, ^{
        self.target = [NSString stringWithFormat:@"ksddkjalkjd%d",i];
    });
}
```

## And
[招聘一个靠谱的iOS-Sunnyxx](https://github.com/ChenYilong/iOSInterviewQuestions/blob/master/01%E3%80%8A%E6%8B%9B%E8%81%98%E4%B8%80%E4%B8%AA%E9%9D%A0%E8%B0%B1%E7%9A%84iOS%E3%80%8B%E9%9D%A2%E8%AF%95%E9%A2%98%E5%8F%82%E8%80%83%E7%AD%94%E6%A1%88/%E3%80%8A%E6%8B%9B%E8%81%98%E4%B8%80%E4%B8%AA%E9%9D%A0%E8%B0%B1%E7%9A%84iOS%E3%80%8B%E9%9D%A2%E8%AF%95%E9%A2%98%E5%8F%82%E8%80%83%E7%AD%94%E6%A1%88%EF%BC%88%E4%B8%8A%EF%BC%89.md#12-arc%E4%B8%8B%E4%B8%8D%E6%98%BE%E5%BC%8F%E6%8C%87%E5%AE%9A%E4%BB%BB%E4%BD%95%E5%B1%9E%E6%80%A7%E5%85%B3%E9%94%AE%E5%AD%97%E6%97%B6%E9%BB%98%E8%AE%A4%E7%9A%84%E5%85%B3%E9%94%AE%E5%AD%97%E9%83%BD%E6%9C%89%E5%93%AA%E4%BA%9B)