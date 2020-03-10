# 微信MLeaksFinder源码阅读



## NSObject+MemoryLeak它的职责就是处理是否触发内存泄漏的逻辑，

```
- (BOOL)willDealloc {
    NSString *className = NSStringFromClass([self class]);
    if ([[NSObject classNamesWhitelist] containsObject:className])
        return NO;// 如果是白名单类，就跳过检测内存泄漏
    
    NSNumber *senderPtr = objc_getAssociatedObject([UIApplication sharedApplication], kLatestSenderKey);
    if ([senderPtr isEqualToNumber:@((uintptr_t)self)])
        return NO;// 如果是正在和用户交互，并且接受事件触摸的，也是跳过
    
    __weak id weakSelf = self;
    dispatch_after(dispatch_time(DISPATCH_TIME_NOW, (int64_t)(2 * NSEC_PER_SEC)), dispatch_get_main_queue(), ^{
        __strong id strongSelf = weakSelf;
        [strongSelf assertNotDealloc];// 2秒后，开启判断对象是否销毁
    });
    
    return YES;
}


```

来到 assertNotDealloc 方法 MLeakedObjectProxy的职责是用NSSet来储存2秒后还没销毁的对象地址（保存对象的地址不会造成内存泄漏）



```
- (void)assertNotDealloc {
    if ([MLeakedObjectProxy isAnyObjectLeakedAtPtrs:[self parentPtrs]]) {
        return;// 如果已经添加到NSSet，就停止，不重复添加
    }
    [MLeakedObjectProxy addLeakedObject:self];// 添加到NSSet里面去
    
    NSString *className = NSStringFromClass([self class]);
    NSLog(@"Possibly Memory Leak.\nIn case that %@ should not be dealloced, override -willDealloc in %@ by returning NO.\nView-ViewController stack: %@", className, className, [self viewStack]);
}
```

来到addLeakedObject方法，它主要是创建一个MLeakedObjectProxy和有内存泄漏的对象进行一一对应映射。并且代理对象弱引用持有内存泄漏对象，而内存泄漏对象强引用持有代理对象。



```
+ (void)addLeakedObject:(id)object {
    NSAssert([NSThread isMainThread], @"Must be in main thread.");
    
    MLeakedObjectProxy *proxy = [[MLeakedObjectProxy alloc] init];// 创建一个代理对象
    proxy.object = object;// 代理对象弱引用持有 内存泄漏的对象
    proxy.objectPtr = @((uintptr_t)object);// 地址
    proxy.viewStack = [object viewStack];// 类名
    static const void *const kLeakedObjectProxyKey = &kLeakedObjectProxyKey;
    objc_setAssociatedObject(object, kLeakedObjectProxyKey, proxy, OBJC_ASSOCIATION_RETAIN);// 内存泄漏强引用只有对象
    
    [leakedObjectPtrs addObject:proxy.objectPtr];
    
#if _INTERNAL_MLF_RC_ENABLED
    [MLeaksMessenger alertWithTitle:@"Memory Leak"
                            message:[NSString stringWithFormat:@"%@", proxy.viewStack]
                           delegate:proxy
              additionalButtonTitle:@"Retain Cycle"];
#else
    [MLeaksMessenger alertWithTitle:@"Memory Leak"
                            message:[NSString stringWithFormat:@"%@", proxy.viewStack]];
#endif
}
```

所以当内存泄漏的对象销毁的时候，内向泄漏对象，就会把它强引用持有的代理对象销毁，这时和内存泄漏对象的隐射代理对象就会销毁，会调用dealloc，这时就把这个内存泄漏地址的地址，从NSSet中移除。



```
- (void)dealloc {
    NSNumber *objectPtr = _objectPtr;
    NSArray *viewStack = _viewStack;
    dispatch_async(dispatch_get_main_queue(), ^{
        [leakedObjectPtrs removeObject:objectPtr];
        [MLeaksMessenger alertWithTitle:@"Object Deallocated"
                                message:[NSString stringWithFormat:@"%@", viewStack]];
    });
}
```

到这里，我们就明白了，从内存泄漏的逻辑判断，到内存泄漏的提示和保存，还有内存泄漏对象它释放后，也会检测得到。

接下来，我们再看，它是如何开始触发内存泄漏计算的，

```
[self swizzleSEL:@selector(pushViewController:animated:) withSEL:@selector(swizzled_pushViewController:animated:)];
        [self swizzleSEL:@selector(popViewControllerAnimated:) withSEL:@selector(swizzled_popViewControllerAnimated:)];
        [self swizzleSEL:@selector(popToViewController:animated:) withSEL:@selector(swizzled_popToViewController:animated:)];
        [self swizzleSEL:@selector(popToRootViewControllerAnimated:) withSEL:@selector(swizzled_popToRootViewControllerAnimated:)];
```

```
[self swizzleSEL:@selector(viewDidDisappear:) withSEL:@selector(swizzled_viewDidDisappear:)];
        [self swizzleSEL:@selector(viewWillAppear:) withSEL:@selector(swizzled_viewWillAppear:)];
        [self swizzleSEL:@selector(dismissViewControllerAnimated:completion:) withSEL:@selector(swizzled_dismissViewControllerAnimated:completion:)];
```

从上面的swizzleSEL逻辑可以看到，它就是检测页面消失触发内存泄漏计算的。然后走，内存泄漏逻辑。

```
[self swizzleSEL:@selector(setView:) withSEL:@selector(swizzled_setView:)];
```

```
[self swizzleSEL:@selector(sendAction:to:from:forEvent:) withSEL:@selector(swizzled_sendAction:to:from:forEvent:)];
```

从上面的swizzleSEL逻辑可以看到，这个就是会一直标记最后一个的对象，它不会走内存泄漏逻辑。



整个库的大致逻辑，就是分三部分。

1. 内存泄漏的逻辑计算。---->是2秒后进入判断

2. 内存泄漏的逻辑变更和如何保存内向泄漏的对象。---->使用一个NSSet装有内存泄漏对象的地址，

   然后创建一个代理和内存泄漏对象一一隐射，当内存泄漏对象销毁时，会调用代理的dealloc

3. 内向泄漏逻辑，是怎么触发的。是页面消失的时候，---->会触发