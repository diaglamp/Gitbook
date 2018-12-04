# 编译优化

## 基础理论

IDE将编译和链接的过程一步完成，这个过程称为构建build

1. 预编译 (Prepressing) - 生成 `.i` 中间文件
   
    - 删除所有 `#define`，展开宏定义
    - 处理所有条件预编译指令，如 `#if` `#ifdef` `#elif` `#else` `#endif`
    - 处理 `#include` 预编译指令，将被包含文件递归插入改指令的位置
    - 删除所有注释
    - 添加行号和文件名标识
    - 保留所有 `#pragma` 指令
  
2. 编译 (Compilation) - 生成 `.s` 汇编文件
   
    - 词法分析 - 扫描器生成记号 `Scanner -> Token`
    - 语法分析 - 语法分析器生成语法树 `Grammer Parser -> Syntax Tree`
    - 语义分析 - 语义分析器为表达式标记类型 `Semantic Analyzer -> Type`
    - 中间语言生成 - 源码优化器生成中间代码 `Source Code Optimizer -> Intermediate Representation`
    - ====== 以上为编译器前端，以下为编译器后端 ======
    - 代码生成器 - 生成汇编代码 `Code Generator -> Assembler`
    - 汇编 - 汇编器生成目标代码 `Assembler -> Object`
    - 目标代码优化 - 目标代码优化器：选择合适的寻址方式、使用位移替代乘法运算、删除多余指令等
  
3. 汇编 (Assembly) - 生成 `.o` 目标文件
   
    - 汇编器将汇编代码转变成机器能够执行的指令。
    - 一般在说编译器的工作的时候会包含汇编的步骤，所以汇编的工作在编译中也包含了
    
4. 链接 (Linking) - 生成 `.out` 可执行文件
  
    - 编译是针对单一文件，如果文件使用了外部的函数或者变量（统称符号Symbol）时，无法知道Symbol的地址，会置为零。
    - 链接时修正外部Symbol的地址，这个过程称为 重定位Relocation

### 静态链接

- 空间与地址分配
    - 多个 .o 合并成一个 .out,代码段数据段合并。
- 符号解析与重定位
    - 目标文件生成时
        - 使用相对地址代表符号
        - 相对地址跟寻址方式相关
    - 重定位表 （目标文件中）
        - 相对地址
        - 符号名称
    - 符号解析
        - 符号名称应该能在全局符号表中找到，否则链接器会报符号未定义错误
    - 指令修正方式
        - 绝对寻址修正
        - 相对寻址修正
- 弱符号处理
    - 链接器支持强符号与弱符号同时存在（强会覆盖弱）
    - 处理多个类型不一致的弱符号问题
    - 选择占空间最大的 Common 块
- 静态库的链接
    - 静态库 `.a` 是一组目标文件的压缩集合

### 目标文件

- 目标文件中符号使用相对地址标记
- 目标文件中有 重定位表

## iOS理论

### iOS 构建流程

- 编译信息写入辅助文件，创建编译后的文件架构（name.app）
- 处理文件打包信息（entitlements）
- 执行编译前脚本
- 编译文件 `CompileC` `clang`
- 链接Framework
- 编译 xib 文件
- 拷贝xib、图片等资源
- 编译 ImageAssets
- 处理info.plist
- 创建 .app 并签名

### 编译过程 

`clang -ccc-print-phases -framework Foundation Demo.m -o Demo`

```
% clang -ccc-print-phases hello.m
0: input, "hello.m", objective-c
1: preprocessor, {0}, objective-c-cpp-output
2: compiler, {1}, assembler
3: assembler, {2}, object
4: linker, {3}, image
5: bind-arch, "x86_64", {4}, image
```

编译器把 Objective-C 代码转化成低级代码，以及对代码做分析，确保代码中没有任何明显的错误。
- 预处理
    - 处理源文件中的宏定义 `#import`
    - 替换自定义宏 `#define`
- 语法与语义分析
    - 词法解析标记 -- 把代码文本从 string 转化成特殊的标记流。
    - 解析 -- 把之前生成的标记流将会被解析成一棵抽象语法树 AST
    - 静态分析 -- 对AST做分析处理，找出错误
- 代码生成
    - 将 AST 转换为更低级的中间码 (LLVM IR)
    - 对生成的中间码做优化
    - 生成特定目标代码
    - 输出汇编代码
- 汇编器
    - 将汇编代码转换为目标对象文件。
- 链接器
    - 将多个目标对象文件合并为一个可执行文件 (或者一个动态库)
    - 解决了目标文件和库之间的链接。

### 编译器职能

- Clang 编译器前端
    - 语法分析
    - 语义分析
    - 生成中间代码
- LLVM (Low Level virtual machine) 编译器后端
    - 机器无关代码优化
    - 生成机器语言
    - 机器相关代码优化
    - BitCode生成
    - 链接期优化
    - 针对不同架构生成不同机器码


### Symbol 符号表

- 每个函数、全局变量和类等都是通过符号的形式来定义和使用的
- 编译后的符号都以地址来访问
- 可执行文件和目标文件有一个符号表，这个符号表规定了它们的符号。
- 例如 `printf` 用地址 `0x100000e80` 来表示


## 实际优化

### 概念

- 增量编译
- 全量编译

### 编译时间监控

1. 方案1
    - `defaults write com.apple.dt.Xcode ShowBuildOperationDuration -bool YES`
    - build clean 后能看到总时间

2. 方案2
    - `gem install xcpretty`  //安装xcpretty软件
    - `npm install -g gnomon` //安装 gnomon
    - `xcodebuild build | xcpretty | gnomon | sort -nr -k1` 后得到时间排序

3. 方案3
    - Xcode10 后能直接看到每个文件的编译时间，但信息太乱
    - 命令行可以选项开启

### 优化方法

- 编译器选项优化
  
    - Debug模式下，不生成dsym文件
    - 将Build Active Architecture Only改为Yes
    - Debug模式下，关闭编译器优化
    - 开启Xcode10 的 `parallel builds `

- 代码层面优化
  
    - forward declaration, 头文件中尽量使用 @class 而不是 #import
    - 移除无效 import
    - 将大的框架抽取为Pods，并打成静态库 .a
    - 经常修改的文件在PCH中移除，减少全量编译的概率

## 参考文章

- [编译器 - objc.io](https://www.objccn.io/issue-6-2/)
- [如果将iOS打包速度提升十倍以上 - bestswifter](https://bestswifter.com/improve_compile_speed/)
- [iOS编译过程的原理和应用](https://github.com/LeoMobileDeveloper/Blogs/blob/master/iOS/iOS%E7%BC%96%E8%AF%91%E8%BF%87%E7%A8%8B%E7%9A%84%E5%8E%9F%E7%90%86%E5%92%8C%E5%BA%94%E7%94%A8.md)

[Building Faster in Xcode - WWDC2018](https://developer.apple.com/videos/play/wwdc2018/408/)