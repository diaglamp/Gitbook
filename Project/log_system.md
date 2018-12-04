# 日志收集

## 解决问题

- 当前日志混乱
- 用户反馈问题跟踪
  
## 解决方案

- 日志分级
- 日志加 tag
- 日志写文件
- 日志上传

## 目标

- 高性能高压缩率
- 不丢失任何一行日志
- 避免系统卡顿
- 避免 CPU 波峰

## 如何写文件

### 对每一行日志加密写文件

磁道寻址由于机械操作的原因相对非常耗时

当写文件的时候，并不是把数据直接写入了磁盘，而是先把数据写入到系统的缓存(dirty page)中，系统一般会在下面几种情况把 dirty page 写入到磁盘：

- 定时回写
- 调用 write 的时候，发现 dirty page 占用内存超过系统内存一定比例
- 内存不足。

数据从程序写入到磁盘的过程中，其实牵涉到两次数据拷贝：一次是把数据从用户态拷贝到内核态，需要经历上下文切换；还有一次是内核空间到硬盘上真正的数据拷贝。当发生回写时也涉及到了内核空间和用户空间频繁切换。 dirty page 回写的时机对应用层来说又是不可控的，所以性能瓶颈就出现了。

### 把日志写入到作为 log 中间 buffer 的内存中，达到一定条件后压缩加密写进文件。

- Crash 的时候会丢日志
- 会有 CPU 峰值

### MMAP

mmap 是使用逻辑内存对磁盘文件进行映射，中间只是进行映射没有任何拷贝操作，避免了写文件的数据拷贝。操作内存就相当于在操作文件，避免了内核空间和用户空间的频繁切换。 

mmap 回写时机

- 内存不足
- 进程 crash
- 调用 msync 或者 munmap

## 压缩和加密

先压缩再加密

为了减少 mmap 所占用内存大小，每一条日志压缩、加密再写入内存。

流式压缩 zlib：在滑动历史缓存中压缩后输出压缩正文。

加密 AES-128 cbc


## Logan 源码分析

Logan 数据协议

- mmap metadata 
    - header + lenth + tail - mmap 元数据长度
    - version - mmap版本
    - path - mmap 数据对应的文件名
- log metadata
    - 四个字节 - 日志数据长度
- log
    - 日志数据

clogan_init

```
aes_init_key_iv
makedir_clogan
open_mmap_file_clogan   150K
read_mmap_data_clogan // 解析 MMAP 数据流 mmap version、mmap path
write_mmap_data_clogan //解析 日志数据 -> 内存模型
```