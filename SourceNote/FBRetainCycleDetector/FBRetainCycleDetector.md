# FBRetainCycleDetector源码阅读



```
__strong 和 __attribute__((objc_precise_lifetime)) 的区别
__strong 是会被编译器优化的，可能会提前release
__attribute__((objc_precise_lifetime)) 是不接受编译器优化的强引用，严格的持有和释放
```

https://clang.llvm.org/docs/AutomaticReferenceCounting.html#precise-lifetime-semantics



测试Block 的C++实现

```
int main(int argc, char * argv[]) {
    void (^aa)(void) = ^void(void){
        int abc = 0;
    };
    [aa copy];
    aa();
    return 1;
}
```

```
xcrun --sdk macosx clang -rewrite-objc ./main.m
```

编译后的block C++代码

```
struct __main_block_impl_0 { // Block的结构
  struct __block_impl impl;
  struct __main_block_desc_0* Desc;
  __main_block_impl_0(void *fp, struct __main_block_desc_0 *desc, int flags=0) {
    impl.isa = &_NSConcreteStackBlock;
    impl.Flags = flags;
    impl.FuncPtr = fp;
    Desc = desc;
  }
};

static void __main_block_func_0(struct __main_block_impl_0 *__cself) {
				// block的回到函数
        int abc = 0;
    }

static struct __main_block_desc_0 {
  size_t reserved;
  size_t Block_size;// block的大小
} __main_block_desc_0_DATA = { 0, sizeof(struct __main_block_impl_0)};


int main(int argc, char * argv[]) {
    void (*aa)(void) = ((void (*)())&__main_block_impl_0((void *)__main_block_func_0, &__main_block_desc_0_DATA)); // 创建Block结构体
    ((id (*)(id, SEL))(void *)objc_msgSend)((id)aa, sel_registerName("copy")); // 发送copy函数
    ((void (*)(__block_impl *))((__block_impl *)aa)->FuncPtr)((__block_impl *)aa); // 通过block结构体对象，调用Block的回到函数
    return 1;
}
```

