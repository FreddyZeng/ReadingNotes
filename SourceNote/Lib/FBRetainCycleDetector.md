# FBRetainCycleDetector源码阅读

## MLeaksFinder获取对象的循环引用

```
MLeaksFinder中当点击控制器的内存泄漏alert弹窗的确定按钮后，就会使用FBRetainCycleDetector去检测当前对象的循环引用的可能性

#pragma mark - UIAlertViewDelegate

- (void)alertView:(UIAlertView *)alertView clickedButtonAtIndex:(NSInteger)buttonIndex {
    ...
    
#if _INTERNAL_MLF_RC_ENABLED
    dispatch_async(dispatch_get_global_queue(0, 0), ^{
        FBRetainCycleDetector *detector = [FBRetainCycleDetector new];
        [detector addCandidate:self.object];
        NSSet *retainCycles = [detector findRetainCyclesWithMaxCycleLength:20];
        
        BOOL hasFound = NO;
        for (NSArray *retainCycle in retainCycles) {
            NSInteger index = 0;
            for (FBObjectiveCGraphElement *element in retainCycle) {
                if (element.object == object) {
                    // 找到了循环引用，是当前的控制器；
                }
                
                ++index;
            }
            if (hasFound) {
                break;
            }
        }
    });
#endif
}
```

## FBRetainCycleDetector的获取循环引用API

```
 [detector addCandidate:self.object]; // 是添加到要寻找循环引用的根节点
 
 - (void)addCandidate:(id)candidate
{
  FBObjectiveCGraphElement *graphElement = FBWrapObjectGraphElement(nil, candidate, _configuration);// 会对传进来的对象包装一层，因为对象可能是Block，也可能是NSObject对象，或者NSTimer，对他们的处理进行抽象。因为具体的获取它们的强引用具体实现不太一样
  if (graphElement) {
    [_candidates addObject:graphElement];
  }
}
```

## findRetainCyclesWithMaxCycleLength 使用DFS深度遍历对象强引用属性，并且检测是否有环

```
NSSet *retainCycles = [detector findRetainCyclesWithMaxCycleLength:20]; // 使用深度优先遍历树DFS

findRetainCyclesWithMaxCycleLength 它的代码太长了。
大概逻辑就是：
_objectSet用来保存过已经遍历计算过的对象，它只保存实例对象的地址。

objectsOnPath 用来保存，当前DFS深度遍历的路径。并且配合_objectSet来防止重复遍历同一个对象的强引用属性。

stack 使用数组来模拟栈，来进行深度遍历的容器。
FBNodeEnumerator *firstAdjacent = [top nextObject];
objectsOnPath 在stack操作中，起到控制作用。
获取top栈顶的nextObject，firstAdjacent!= null,如果 retainCycles 包含 retainCycles，存在环，也就是循环引用。添加到 retainCycles。如果不存在环，就入栈，下次节点遍历从top开始，也就是深度优先。（所以，当找到循环引用的对象，就不会再入栈，这样就不会无限循环了）

如果objectsOnPath已经包含过这个对象[objectsOnPath containsObject:firstAdjacent]，就跳过。如果firstAdjacent == null，说明当前的栈顶对象的属性都遍历完了，出栈。

```



## nextObject迭代器模式

```
- (FBNodeEnumerator *)nextObject
{
  if (!_object) {
    return nil;
  } else if (!_retainedObjectsSnapshot) {
    _retainedObjectsSnapshot = [_object allRetainedObjects];
    _enumerator = [_retainedObjectsSnapshot objectEnumerator];
  }

  FBObjectiveCGraphElement *next = [_enumerator nextObject];// 其实就是对 allRetainedObjects获取到的所有强引用对象进行NSSet的迭代，也就是遍历 _object 的强引用属性

  if (next) {
    return [[FBNodeEnumerator alloc] initWithObject:next];
  }

  return nil;
}
```

## allRetainedObjects是FBObjectiveCGraphElement的抽象方法

FBObjectiveCGraphElement有三个具体实现类，FBObjectiveCBlock、FBObjectiveCObject和FBObjectiveCNSCFTimer

FBAssociationManager它使用了fish Hook了系统的关联对象的相关两个函数。
allRetainedObjects 在 FBObjectiveCGraphElement 中的主要职责是，使用 FBAssociationManager获取关联对象的强引用，并且使用过滤和转换 返回给子类的强引用逻辑，同时根据object是block还是NSTimer对象，还是普通OC对象，生成对应的FBObjectiveCGraphElement子类。

```
- (NSSet *)allRetainedObjects
{
  NSArray *retainedObjectsNotWrapped = [FBAssociationManager associationsForObject:_object];
  NSMutableSet *retainedObjects = [NSMutableSet new];

  for (id obj in retainedObjectsNotWrapped) {
    FBObjectiveCGraphElement *element = FBWrapObjectGraphElementWithContext(self,
                                                                            obj,
                                                                            _configuration,
                                                                         @[@"__associated_object"]);
```

FBObjectiveCBlock是为了获取Block对象的所有强引用，FBObjectiveCObject是获取所有OC对象的强引用，FBObjectiveCNSCFTimer是为了获取所有定时器NSTimer的强引用。

我们先从Block开始看看它是怎么获取到强引用的吧。这里的Block，是指NSObject作为成员变量持有的Block闭包。

## FBObjectiveCBlock Block对象的强引用获取

```
- (NSSet *)allRetainedObjects
{
  NSMutableArray *results = [[[super allRetainedObjects] allObjects] mutableCopy];

  // Grab a strong reference to the object, otherwise it can crash while doing
  // nasty stuff on deallocation
  __attribute__((objc_precise_lifetime)) id anObject = self.object;

  void *blockObjectReference = (__bridge void *)anObject;
  NSArray *allRetainedReferences = FBGetBlockStrongReferences(blockObjectReference);

  for (id object in allRetainedReferences) {
    FBObjectiveCGraphElement *element = FBWrapObjectGraphElement(self, object, self.configuration);
    if (element) {
      [results addObject:element];
    }
  }

  return [NSSet setWithArray:results];
}
```

从上面可以看到，这个是使用禁止编译器优化的强引用，[防止对象被编译器优化](https://clang.llvm.org/docs/AutomaticReferenceCounting.html#precise-lifetime-semantics)

```
__strong 和 __attribute__((objc_precise_lifetime)) 的区别
__strong 是会被编译器优化的，可能会提前release
__attribute__((objc_precise_lifetime)) 是不接受编译器优化的强引用，严格的持有和释放
```

FBGetBlockStrongReferences ---> _GetBlockStrongLayout

```
NSArray *FBGetBlockStrongReferences(void *block) {
  if (!FBObjectIsBlock(block)) {
    return nil;
  }
  
  NSMutableArray *results = [NSMutableArray new];

  void **blockReference = block;
  NSIndexSet *strongLayout = _GetBlockStrongLayout(block);// 获取到强引用Block的对象的内存布局
  [strongLayout enumerateIndexesUsingBlock:^(NSUInteger idx, BOOL *stop) {
    void **reference = &blockReference[idx];使用Block的指针，进行idx偏移，获取到Block持有的强引用指向

    if (reference && (*reference)) {
      id object = (id)(*reference);// 解Block的指针，取得的值，就是Block强引用持有对象的地址

      if (object) {
        [results addObject:object];
      }
    }
  }];

  return [results autorelease];
}
```

_GetBlockStrongLayout 函数主要的职责，就是判断Block的类型，只有BLOCK_HAS_COPY_DISPOSE，有help函数并且Block不是捕抓到持有C++对象的Block 才能使用模拟Block的内存布局去测试获取Block的强引用指针。（因为Block捕抓C++对象后，内存布局没有对齐并且C++的对象生成的help函数是调用C++的析构函数，而Block捕抓普通OC对象后，它的指针是在Block的结构体后面，并且可以看到编译器生成的help释放内存函数是传第一个block捕抓的对象指针就可以的了，[这有详细的Block内存布局实现](http://clang.llvm.org/docs/Block-ABI-Apple.html)）

 主要逻辑，创建两个数组来存放连续的虚假对象，并且后面会调用dispose_helper，模拟释放obj数组。
虚假对象会覆盖Release函数，当它被调用的时候，会设置isStrong = YES；来标记这个对象被block的dispose_helper处理了。所以就能检查出Block的强引用在内存布局的那个位置。

```
static NSIndexSet *_GetBlockStrongLayout(void *block) {
  struct BlockLiteral *blockLiteral = block;

  /**
   BLOCK_HAS_CTOR - Block has a C++ constructor/destructor, which gives us a good chance it retains
   objects that are not pointer aligned, so omit them.

   !BLOCK_HAS_COPY_DISPOSE - Block doesn't have a dispose function, so it does not retain objects and
   we are not able to blackbox it.
   */
  if ((blockLiteral->flags & BLOCK_HAS_CTOR)
      || !(blockLiteral->flags & BLOCK_HAS_COPY_DISPOSE)) {
    return nil;
  }

  void (*dispose_helper)(void *src) = blockLiteral->descriptor->dispose_helper;
  const size_t ptrSize = sizeof(void *);

  // Figure out the number of pointers it takes to fill out the object, rounding up.
  // 获取block的结构体大小
  const size_t elements = (blockLiteral->descriptor->size + ptrSize - 1) / ptrSize;

  // Create a fake object of the appropriate length.
    // 创建两个连续的虚假对象，并且后面会调用dispose_helper，模拟释放obj
  void *obj[elements];
  void *detectors[elements];

  for (size_t i = 0; i < elements; ++i) {
    FBBlockStrongRelationDetector *detector = [FBBlockStrongRelationDetector new];
    obj[i] = detectors[i] = detector;
  }

  @autoreleasepool {
    dispose_helper(obj);
  }

  // Run through the release detectors and add each one that got released to the object's
  // strong ivar layout.
  NSMutableIndexSet *layout = [NSMutableIndexSet indexSet];

  for (size_t i = 0; i < elements; ++i) {
    FBBlockStrongRelationDetector *detector = (FBBlockStrongRelationDetector *)(detectors[i]);
    if (detector.isStrong) {
      [layout addIndex:i];
    }

    // Destroy detectors
    [detector trueRelease];
  }

  return layout;
}
```



### Block闭包释放内存

Block闭包捕抓普通对象释放

```
void __block_dispose_4(struct __block_literal_4 *src) {
     // was _Block_destroy
     _Block_object_dispose(src->existingBlock, BLOCK_FIELD_IS_BLOCK);
}
```

Block闭包捕抓C++对象释放

```
void __block_dispose_10(struct __block_literal_10 *src) {
     FOO_dtor(&src->foo);
}
```

Block闭包捕抓普通OC对象，为什么_Block_object_dispose 就一定会调用OC对象的Release呢？[可以从这里获得它的实现](https://opensource.apple.com/source/libclosure/libclosure-74/runtime.cpp.auto.html)

```
// When Blocks or Block_byrefs hold objects their destroy helper routines call this entry point
// to help dispose of the contents
void _Block_object_dispose(const void *object, const int flags) {
    switch (os_assumes(flags & BLOCK_ALL_COPY_DISPOSE_FLAGS)) {
      case BLOCK_FIELD_IS_BYREF | BLOCK_FIELD_IS_WEAK:
      case BLOCK_FIELD_IS_BYREF:
        // get rid of the __block data structure held in a Block
        _Block_byref_release(object);
        break;
      case BLOCK_FIELD_IS_BLOCK:
        _Block_release(object);
        break;
      case BLOCK_FIELD_IS_OBJECT:
        _Block_release_object(object);// 从这里可以看到，它是会调用Release
        break;
      case BLOCK_BYREF_CALLER | BLOCK_FIELD_IS_OBJECT:
      case BLOCK_BYREF_CALLER | BLOCK_FIELD_IS_BLOCK:
      case BLOCK_BYREF_CALLER | BLOCK_FIELD_IS_OBJECT | BLOCK_FIELD_IS_WEAK:
      case BLOCK_BYREF_CALLER | BLOCK_FIELD_IS_BLOCK  | BLOCK_FIELD_IS_WEAK:
        break;
      default:
        break;
    }
}
```



### Block自行编译生成C++代码

可以自行测试编译Block 的代码，生成C++的实现 [这有详细的实现](http://clang.llvm.org/docs/Block-ABI-Apple.html)

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

## FBObjectiveCObject  普通OC对象的强引用获取



## FBObjectiveCNSCFTimer  普通OC对象的强引用获取

```
typedef struct {
  long _unknown; // This is always 1
  id target;
  SEL selector;
  NSDictionary *userInfo;
} _FBNSCFTimerInfoStruct;

- (NSSet *)allRetainedObjects
{
  // Let's retain our timer
  __attribute__((objc_precise_lifetime)) NSTimer *timer = self.object;

  if (!timer) {
    return nil;
  }

  NSMutableSet *retained = [[super allRetainedObjects] mutableCopy];

  CFRunLoopTimerContext context;
  CFRunLoopTimerGetContext((CFRunLoopTimerRef)timer, &context);

  // If it has a retain function, let's assume it retains strongly
  if (context.info && context.retain) {
    _FBNSCFTimerInfoStruct infoStruct = *(_FBNSCFTimerInfoStruct *)(context.info);// 为什么这么确定这个结构体的内存布局呢？我查了runtime源码，没发现这个info，就一定是上面的结构体。它有可能是个函数。
    if (infoStruct.target) {
      FBObjectiveCGraphElement *element = FBWrapObjectGraphElementWithContext(self, infoStruct.target, self.configuration, @[@"target"]);
      if (element) {
        [retained addObject:element];
      }
    }
    if (infoStruct.userInfo) {
      FBObjectiveCGraphElement *element = FBWrapObjectGraphElementWithContext(self, infoStruct.userInfo, self.configuration, @[@"userInfo"]);
      if (element) {
        [retained addObject:element];
      }
    }
  }

  return retained;
}
```

从上面看到主要是从runloop中获取CFRunLoopTimerGetContext的context，然后去到info结构体，再去到target和字典信息

真的想知道这个info，为什么是这样的布局。我们只能够创建一个demo创建定时器，并且通过LLDB 正则断点CFRunLoopTimerContext，然后查看这个定时器context的内存值，看看是不是定时器实例对象的地址了。很多时候，看源码，要看懂，然后再问自己，为什么作者可以这么写，他是根据什么得出来的。这样可以拓展我们的思维，以后遇到类似问题，也可以参考。