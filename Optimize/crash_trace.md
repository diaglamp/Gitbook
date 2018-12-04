# Crash

## 异常种类

### Mach 异常

Mach异常是最底层的内核级异常

- EXC_BAD_ACCESS （内存访问异常）
- EXC_BAD_INSTRUCTION （Illegal or undefined instruction or operand）

详细类型可查看 CocoaSDK 中 `mach/exception_types.h` 文件

### Unix Singal

Unix Signal是Unix系统中的一种异步通知机制，Mach异常在host层被ux_exception转换为相应的Unix Signal，并通过threadsignal将信号投递到出错的线程。

- SIGTRAP 5 (由断点指令或其它trap指令产生. 由debugger使用)
- SIGABRT 6 (调用abort函数生成的信号)
- SIGBUS 7 (非法地址, 包括内存地址对齐出错)
- SIGSEGV 11 （试图访问未分配给自己的内存, 或试图往没有写权限的内存地址写数据.）
- SIGALRM 14（程序超时信号)

利用Unix标准的signal机制来处理异常。详细类型可查看 `sys/signal.h`

### NSException

NSException是OC层，由iOS库或者各种第三方库或Runtime验证出错误而抛出的异常。

- NSRangeException （数组越界异常）

它们可以被try catch捕获（苹果不建议用），如果未被捕获或被@throw抛出，可以通过注册NSSetUncaughtExceptionHandler函数来捕获处理。

## 异常流程

- 当错误发生时候，先在最底层产生Mach异常
- Mach异常在host层被转换为相应的Unix Signal
- 在OC层如果有对应的NSException（OC异常），就转换成OC异常
- OC异常可以在OC层得到处理
- 如果OC异常一直得不到处理，程序会强行发送SIGABRT信号中断程序
- 如果OC层没有对应的NSException，就只能让Unix标准的signal机制来处理了。

## Crash 日志组成

- header
    - 手机型号
    - 进程名称
    - appID
    - app 版本
    - 系统版本
    - 崩溃时间
- 异常信息
    - Exception Type: EXC_BAD_ACCESS (SIGSEGV)
    - Exception Subtype: KERN_INVALID_ADDRESS at 0x0000000000000000
    - Exception Codes: 0x0000000000000000
    - Termination Signal: Segmentation fault: 11
    - Termination Reason: Namespace SIGNAL
- 线程回溯
    - 崩溃堆栈
- 线程状态
- 二进制映像