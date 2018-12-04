# 启动优化

## 理论

Virtual Memory 虚拟内存

- 中间层
- 逻辑地址 - 物理地址 的映射
- 逻辑地址不映射物理RAM， 访问时会 page fault
- 多个逻辑地址映射到同一个物理RAM ， 多进程共享内存
- clean / dirty page
- mmap（）
- Copy-On-Write, COW
  
Mach-O Image Loading

- __Text and __LINKEDIT 内存共享
- __DATA dirty page, COW
- __LINKEDIT 用完回收
  
安全

- ASLR (Address Space Layout Randomization)：地址空间布局随机化，镜像会在随机的地址上加载。
- 代码签名：把每页内容都生成一个单独的加密散列值，并存储在 `__LINKEDIT` 中
  
 exec() to main()

- exec() 系统调用.
- Kernel maps your app info new address space
- Start of your app is random
- Low memory is marked inaccessible
  
Dylibs

- Dyld : helper program to load dylibs 
- Dyld runs in progress
- 内核完成映射进程的工作后将 dyld 的 Mach-O 映射到进程中的随机地址，并将PC寄存器设为 dyld 的地址并运行。
  
 Dyld Steps

- Load Dylibs image
- Rebase Image
- Bind Image
- ObjC setup
- Initializers

加载一个动态库的过程

- 读取主可执行文件的头文件，内核已经映射好
- Parse list of dependent dylibs
- Find requested mach-o file
- Open and read start of file
- valid mach-o
- Register code signature
- Call mmap() for each segment

### 虚拟地址空间 Virtual Address Space

- 程序 - 静态 - 可执行文件 - 预先编译好的指令和数据集合的文件
- 进程 - 动态 - 程序运行时的过程
- 程序运行后，有自己独立的虚拟地址空间
- 由CPU位数决定，32位CPU最多支持4G， 64位CPU最多支持 17179869184G
- 一般来说，C语言指针大小的位数与虚拟空间的位数相同
- 可访问空间由操作系统分配
- 如果访问了未经允许的空间，操作系统会捕获访问，抛出异常（crash），错误原因为 Segmentation fault
- 操作系统使用 0xC0000000 - 0xFFFFFFFF 1GB, 其他原则上分配给进程


### Building Mach-O Files

- Products
    - Types of Mach-O files you can build
    - The main executable file usually contains the core logic of the program, including the entry point main function.
    - Intermediate object files. 
    - Dynamic shared libraries.
    - Frameworks
    - Umbrella frameworks. 
    - Static archive libraries.
    - Bundles.
    - Kernel extensions.

- Module
    - The Smallest Unit of Code
    - compiler generate output object file is a module, main.o
    - The static linker can reduce several input modules into a single module. 

- Static Archive Libraries
    - different from Mach-O file
    - Group a set of modules
    - An archive file with a table of contents entry.
    - MacOS libtool `ar command`
    - The typical purpose is to group several object files together with a table of contents
    - The static linker can link the object files stored in a static archive library into a Mach-O executable or dynamic library. 

### Executing Mach-O Files

- Summary
    - Link share dynamic shared libraries.
    - define references to symbols in those modules
    - resolved at runtime.
    - At runtime the symbol names of all the modules share namespace
- Launching an Application
    - call Func `fork` and `execve`
    - fork creates a process
    - execve loads and executes the program
- Finding Imported Symbols
    - Binding Symbols
        - Binding is the process of resolving a module’s references to functions and data in other modules
        - The modules may be in the same Mach-O file or in different Mach-O files
    - Searching for Symbols
        - A symbol is a generic representation of the location of a function, data variable, or constant in an executable file.

### Loading Code at Runtime


### Indirect Addressing

- Summary
    - co-gen - code generation technique
    - allow symbols defined in one file to be referenced from another file.
    - Indirect addressing minimizes the number of locations that must be modified by the dynamic linker
    - Non-lazy symbol references 
        - resolved by the dynamic linker when a module is loaded.
        - for data symbols or function addresses
    - Lazy symbol references
        - resolved by the dynamic linker the first time they are used (not at load time).
        - made up of a symbol pointer and a symbol stub
        - The compiler generates lazy symbol references when it encounters a call to a function defined in another file.
        - 用于可执行文件中调用不包含在可执行文件中的函数
    - Position-Independent Code

### Position-Independent Code

- Summary
    - PIC, co-gen
    - allows the dynamic linker to load a region of code at a non-fixed virtual memory address.


## 技术方案

- `t(App总启动时间) = t1 + t2`
- `t1 = 系统dylib(动态链接库)和自身App可执行文件的加载`
- `t2 = main方法执行之后到AppDelegate类中的 didFinishLaunchingWithOptions:方法执行结束前这段时间，主要是构建第一个界面，并完成渲染展示。`

### 系统启动时序

- 加载可执行文件 MachO
- 加载动态链接库 dyld
  
- load dylibs image 读取库镜像文件
    - 分析依赖的动态库
    - 找到动态库的 mach-o 文件
    - 打开文件
    - 验证文件
    - 在系统核心注册文件签名
    - 对动态库的每一个 segment 调用 mmap() 
- Rebase image
    - 修复指向当前镜像内部的资源指针
- Bind image
    - 修复指向的镜像外部的资源指针
- Objc setup
    - 注册Objc类
    - 注册category，插入分类中的方法
    - 保证 selector 唯一
- initializers
    - `+ load ()` 方法
    - C ++ 构造函数
    - 非基本类型 C++ 静态全局变量创建

### 启动调用堆栈

- dyld 初始化程序二进制文件
- ImageLoader 递归读取 image，包括自定义的类、方法等各种符号
- runtime 绑定 dyld 回调，image 加载到内存后，dyld 会通知 runtime 处理
- runtime 调用 map/images 做解析和处理，loadImages中调用 call_load_methods 方法，遍历所有加载进来的Class，按层级关系依次调用 Class 的 +load 和 Category 的 +load

整个事件由 dyld 主导，完成运行环境的初始化后，配合 ImageLoader 将二进制文件按格式加载到内存， 动态链接依赖库，并由 runtime 负责加载成 objc 定义的结构，所有初始化工作结束后，dyld 调用真正的 main 函数。


### 时间统计

- premain
- main

### 可优化点

- 减少不必要的 framework，动态链接比较耗时
- 合并或者删减一些 OC 类，清理项目中无用的类
- 删减无用静态变量
- 删减没有被调用或者废弃的方法
- 将不必须在+load方法中做的事情延迟到+initialize中
- 尽量不要用C++虚函数

- 重点其实都在业务逻辑