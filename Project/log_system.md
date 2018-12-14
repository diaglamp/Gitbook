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

Logan MMAP 缓存数据协议

![mmap](mmap.png)

初始化流程

```c
int clogan_init() {
    aes_init_key_iv()   // 加密初始化
    makedir_clogan() // 创建文件
    open_mmap_file_clogan()   // mmap buffer, memory cache
    read_mmap_data_clogan() // 解析 MMAP metadata (回写上次的mmap)
        write_mmap_data_clogan() // 解析 mmap 日志数据
            init_file_clogan(logan_model) // mmap 日志数据 flush
            clogan_flush()
}
```

打开日志文件，准备写入
```c
int clogan_open() {
    clogan_flush() // 回写日志
    init_file_clogan() // 初始化文件
    init_zlib_clogan() // 初始化 zlib
    add_mmap_header_clogan() // 写入 mmap metadata
}
```

写入日志
```c
int clogan_write() {
    is_file_exist_clogan(file_path) // 校验日志文件
    is_file_exist_clogan(_mmap_file_path) // 校验 mmap 文件
    construct_json_data_clogan() // 拼装 log json
    clogan_write_section() // 写入日志 (>20K 切割写入)
        clogan_write2()
            clogan_zlib_compress() // 日志压缩加密
            update_length_clogan() // 更新 mmap len
            clogan_zlib_end_compress() // 压缩单元结束
            update_length_clogan() //
            if(emptyfile || memory || len > mmap) // 写入 log 文件时机
                write_flush_clogan()
}
```

日志写入 log 文件
```c
void write_flush_clogan() {
    write_dest_clogan() //文件写入磁盘、更新文件大小
        is_file_exist_clogan() // 校验日志文件
        insert_header_file_clogan() //空文件插入一行clogan
        fwrite() // 写入 
        fflush() // flush
    clear_clogan()
}
```

logan_model
```c

static cLogan_model *logan_model = NULL; //(不释放)

typedef struct logan_model_struct {
    int total_len; //数据长度
    char *file_path; //文件路径

    // zlib
    int is_malloc_zlib;
    z_stream *strm;
    int zlib_type; //压缩状态
    int is_ready_gzip; //是否可以gzip
    
    // 流式压缩
    char remain_data[16]; //剩余空间
    int remain_data_len; //剩余空间长度

    // 日志文件
    int file_stream_type; //文件流类型
    FILE *file; //文件流
    long file_len; //文件大小

    // 指针
    unsigned char *buffer_point; //mmap缓存的指针
    unsigned char *last_point; //最后写入位置的指针
    unsigned char *total_point; //日志长度的指针
    unsigned char *content_lent_point;//协议内容长度指针 , 给java看,高字节
    int content_len; //内容的大小

    unsigned char aes_iv[16]; //aes_iv
    int is_ok;

} cLogan_model;
```