# Block

## Block 的实现

源码:
```objc
int main()
{
    void (^blk) (void) = ^ {
        printf("heihei");
    };
    blk();
    return 0;
}
```

block 结构体，本质为对象
```objc
struct __block_impl {
  void *isa;
  int Flags;
  int Reserved;
  void *FuncPtr;
};
```

具体的一个 Block 对象，底层实现：

```objc
// 具体的 block 对象，包括 impl、desc 结构体，以及构造函数
struct __main_block_impl_0 {
  struct __block_impl impl;
  struct __main_block_desc_0* Desc;
  __main_block_impl_0(void *fp, struct __main_block_desc_0 *desc, int flags=0) {
    impl.isa = &_NSConcreteStackBlock;
    impl.Flags = flags;
    impl.FuncPtr = fp;
    Desc = desc;
  }
};

// Function Pointer 指向的实际的 block
static void __main_block_func_0(struct __main_block_impl_0 *__cself) {
        printf("heihei");
}

// 
static struct __main_block_desc_0 {
  size_t reserved;
  size_t Block_size;
} __main_block_desc_0_DATA = { 0, sizeof(struct __main_block_impl_0)};
```

调用:
```objc
int main()
{
    void (*blk) (void) = ((void (*)())&__main_block_impl_0((void *)__main_block_func_0, &__main_block_desc_0_DATA));
    ((void (*)(__block_impl *))((__block_impl *)blk)->FuncPtr)((__block_impl *)blk);
    return 0;
}
```

> 命名规则：根据 Block 语法所属的函数名 (main)，以及在该函数出现的顺序 (0)

## 一个 Block 的内存布局

```objc
struct Block_desc {
    size_t reserved;
    size_t Block_size;
}

struct Block_layout {
    void *isa;
    int flags;
    int reserved;
    void *FuncPtr;
    struct Block_desc *desc;
    /* Captured variables. */
    //...
}
```

## 变量捕获

捕获自动变量

``` objc
int a = 100;

// capture 了变量的值
struct __main_block_impl_0 {
  struct __block_impl impl;
  struct __main_block_desc_0* Desc;
  int a;
  __main_block_impl_0(void *fp, struct __main_block_desc_0 *desc, int _a, int flags=0) : a(_a) {
    impl.isa = &_NSConcreteStackBlock;
    impl.Flags = flags;
    impl.FuncPtr = fp;
    Desc = desc;
  }
};
// 在调用实际 func 时赋值
static void __main_block_func_0(struct __main_block_impl_0 *__cself) {
  int a = __cself->a; // bound by copy

        printf("hello %d\n",a);
}
```

捕获静态变量/全局变量/局部静态变量

``` objc
static int a = 100;

// 捕获静态变量的指针
struct __main_block_impl_0 {
  struct __block_impl impl;
  struct __main_block_desc_0* Desc;
  int *a;
  __main_block_impl_0(void *fp, struct __main_block_desc_0 *desc, int *_a, int flags=0) : a(_a) {
    impl.isa = &_NSConcreteStackBlock;
    impl.Flags = flags;
    impl.FuncPtr = fp;
    Desc = desc;
  }
};
// 指针变量赋值
static void __main_block_func_0(struct __main_block_impl_0 *__cself) {
  int *a = __cself->a; // bound by copy

        (*a) = 5;
        printf("hello %d\n",(*a));
    }
```

捕获 __block 变量

``` objc
__block int a = 100;

//__block 修饰变量 -> 对象 
struct __Block_byref_a_0 {
  void *__isa;
__Block_byref_a_0 *__forwarding;
 int __flags;
 int __size;
 int a;
};

//捕获指针
struct __main_block_impl_0 {
  struct __block_impl impl;
  struct __main_block_desc_0* Desc;
  __Block_byref_a_0 *a; // by ref
  __main_block_impl_0(void *fp, struct __main_block_desc_0 *desc, __Block_byref_a_0 *_a, int flags=0) : a(_a->__forwarding) {
    impl.isa = &_NSConcreteStackBlock;
    impl.Flags = flags;
    impl.FuncPtr = fp;
    Desc = desc;
  }
};
//捕获的对象原地址获取变量
static void __main_block_func_0(struct __main_block_impl_0 *__cself) {
  __Block_byref_a_0 *a = __cself->a; // bound by ref

        (a->__forwarding->a) = 5;
        printf("hello %d\n",(a->__forwarding->a));
}
// copy dispose 用于管理捕获对象的内存（局部变量超出范围会释放）
static void __main_block_copy_0(struct __main_block_impl_0*dst, struct __main_block_impl_0*src) {_Block_object_assign((void*)&dst->a, (void*)src->a, 8/*BLOCK_FIELD_IS_BYREF*/);}
static void __main_block_dispose_0(struct __main_block_impl_0*src) {_Block_object_dispose((void*)src->a, 8/*BLOCK_FIELD_IS_BYREF*/);}

static struct __main_block_desc_0 {
  size_t reserved;
  size_t Block_size;
  void (*copy)(struct __main_block_impl_0*, struct __main_block_impl_0*);
  void (*dispose)(struct __main_block_impl_0*);
} __main_block_desc_0_DATA = { 0, sizeof(struct __main_block_impl_0), __main_block_copy_0, __main_block_dispose_0};
```


## 在 Objective-C 语言中，一共有 3 种类型的 block：

- _NSConcreteGlobalBlock 全局的静态 block，不访问外部变量或只访问静态变量。
- _NSConcreteStackBlock 保存在栈中的 block，访问自动变量，当函数返回时会被销毁。
- _NSConcreteMallocBlock 保存在堆中的 block，栈中拷贝到堆区，当引用计数为 0 时会被销毁。


> 上面的例子中,由于 clang 改写的具体实现方式和 LLVM 不太一样，并且这里没有开启 ARC。所以这里我们看到 isa 指向的还是_NSConcreteStackBlock。但在 LLVM 的实现中，开启 ARC 时，block 应该是 _NSConcreteGlobalBlock 类型

## 什么时候 Block 需要在栈区拷贝到堆区，为什么

先了解一下 C 语言的 return 机制，了解一个函数在栈中的生命周期

- 函数压栈
- 如果有成员变量，为成员变量分配栈空间
- 执行到 return 时，将返回值存入寄存器 eax 中
- 函数运行完毕时自动释放在栈中的局部变量
- 函数出栈

这里要区分一下成员变量的类型，基本数据类型、指针、block、对象

基本数据类型：return时已经存入了 eax 中，成员变量释放了也不妨碍上一个函数使用

指针：return时存入的是指针，指针指向的栈内存在函数出栈时已被释放，再访问时会出错。

对象：虽然对象也是成员变量，但超出作用域的时候并不会释放，为什么？因为对象是 malloc 出来的，内存分配在堆区，所以返回的指针指向的地址仍然有效。

block：在 oc 中就是一个对象，但 block 不是 malloc 出来的。block 变量可以理解为指针，问题同上。

