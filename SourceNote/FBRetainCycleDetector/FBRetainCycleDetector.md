# FBRetainCycleDetector源码阅读



```
__strong 和 __attribute__((objc_precise_lifetime)) 的区别
__strong 是会被编译器优化的，可能会提前release
__attribute__((objc_precise_lifetime)) 是不接受编译器优化的强引用，严格的持有和释放
```

https://clang.llvm.org/docs/AutomaticReferenceCounting.html#precise-lifetime-semantics

