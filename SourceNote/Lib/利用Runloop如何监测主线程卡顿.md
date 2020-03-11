# 利用Runloop如何监测主线程卡顿

来源：https://github.com/ming1016/GCDFetchFeed/blob/master/GCDFetchFeed/GCDFetchFeed/Lib/SMLagMonitor/SMLagMonitor.m

Runloop是基于线程的。可以这样简单的理解，检测当前线程是否有任务，如果有任务，就调度CPU，然后执行线程的任务。如果没任务，就不用占用大量的CPU资源。

Runloop状态

```
    kCFRunLoopEntry , //  进入 Runloop
    kCFRunLoopBeforeTimers , // 触发 Timer 回调
    kCFRunLoopBeforeSources , // 触发 Source0 回调
    kCFRunLoopBeforeWaiting , // 等待 mach_port 消息
    kCFRunLoopAfterWaiting ), // 接收 mach_port 消息,此时线程刚才休眠中唤醒，会按处理 Timer Source0
    kCFRunLoopExit , // 退出 loop
```

```
// 通知 Observers RunLoop 会触发 Source0 回调
if (currentMode->_observerMask & kCFRunLoopBeforeSources)
    __CFRunLoopDoObservers(runloop, currentMode, kCFRunLoopBeforeSources);
// 执行 block
__CFRunLoopDoBlocks(runloop, currentMode);
```

## 卡顿检测逻辑

​    kCFRunLoopBeforeTimers , // 触发 Timer 回调

​    kCFRunLoopBeforeSources , // 触发 Source0 回调

​    kCFRunLoopBeforeWaiting , // 等待 mach_port 消息

​    kCFRunLoopAfterWaiting ), // 接收 mach_port 消息,此时线程刚才休眠中唤醒，会按处理 Timer Source1

​	kCFRunLoopExit , // Runloop 可能会在这里 退出

上面Runloop的一个循环流程，所以我们只要测出

所以，我们来看子线程监控的监控和Runloop状态变化的回调kCFRunLoopBeforeTimers -> kCFRunLoopAfterWaiting这个段的时间，就可以测量出一个Runloop循环的时间。

所以，我们来看子线程监控的监控和Runloop状态变化的回调

```
//创建子线程监控
    dispatch_async(dispatch_get_global_queue(0, 0), ^{
        //子线程开启一个持续的loop用来进行监控
        while (YES) {
            long semaphoreWait = dispatch_semaphore_wait(dispatchSemaphore, dispatch_time(DISPATCH_TIME_NOW, STUCKMONITORRATE * NSEC_PER_MSEC));// STUCKMONITORRATE毫秒
            if (semaphoreWait != 0) {
                if (!runLoopObserver) {
                    timeoutCount = 0;
                    dispatchSemaphore = 0;
                    runLoopActivity = 0;
                    return;
                }
                //两个runloop的状态，BeforeSources和AfterWaiting这两个状态区间时间能够检测到是否卡顿
                if (runLoopActivity == kCFRunLoopBeforeSources || runLoopActivity == kCFRunLoopAfterWaiting) {
                    //出现三次出结果
                    if (++timeoutCount < 3) {
                        continue;
                    }
//                    NSLog(@"monitor trigger");
                    dispatch_async(dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_HIGH, 0), ^{
                        NSString *stackStr = [SMCallStack callStackWithType:SMCallStackTypeMain];
                        SMCallStackModel *model = [[SMCallStackModel alloc] init];
                        model.stackStr = stackStr;
                        model.isStuck = YES;
                        [[[SMLagDB shareInstance] increaseWithStackModel:model] subscribeNext:^(id x) {}];
                    });
                } //end activity
            }// end semaphore wait
            timeoutCount = 0;
        }// end while
    });
```

```
static void runLoopObserverCallBack(CFRunLoopObserverRef observer, CFRunLoopActivity activity, void *info){
    SMLagMonitor *lagMonitor = (__bridge SMLagMonitor*)info;
    lagMonitor->runLoopActivity = activity;
    
    dispatch_semaphore_t semaphore = lagMonitor->dispatchSemaphore;
    dispatch_semaphore_signal(semaphore);
}
```

1. 监听子线程可以看做是消费者，Runloop的监测回调可以看做是生产者。
2. 如果生产者一直以比较快的速度生产，那么子线程的信号量是不会等待很久的，所以会重置timeout次数。
3. 如果主线程的状态变化很慢，那么监听的子线程的信号量可能就会超时
4. 如果超时，就会进入超时计算逻辑。

```
long semaphoreWait = dispatch_semaphore_wait(dispatchSemaphore, dispatch_time(DISPATCH_TIME_NOW, STUCKMONITORRATE * NSEC_PER_MSEC));// 等待 STUCKMONITORRATE 毫秒
```

上面的子线程信号量，就是等待STUCKMONITORRATE毫秒，如果超时，那么就会进入 if (semaphoreWait != 0) 分支

timeoutCount是计算超时的次数，如果累计连续3次Runloop事件循环超时。那么我们就认为此时主线程发生了卡顿了。

以上就是大致原理，

1. 主要利用到测量主线程Runloop事件循环的时长是否超过设定的时间。
3. 选用信号量的dispatch_semaphore_wait来设置超时计算逻辑，这样可以很好的计算Runloop的状态，到底停留了多久，而且可以自己修改控制。

在滑动TableView的时候，主Runloop处于UITrackingRunLoopMode，如果监听主线程Runloop是UITrackingRunLoopMode，这个时候我们就不往主线程添加任务，我们就可以尽量避免CPU占用率太高了。可以创建一个队列，累积任务，等主线程Runloop不处于UITrackingRunLoopMode的时候，再添加到主队列中去
