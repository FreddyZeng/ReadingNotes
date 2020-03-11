# Out-Of-Memory大内存检测和定位

资料来源：https://juejin.im/post/5c28646f5188257abf1d947d

## 获取真实的app占用内存

```
#import <mach/mach.h>

- (int64_t)memoryUsage {
    int64_t memoryUsageInByte = 0;
    task_vm_info_data_t vmInfo;
    mach_msg_type_number_t count = TASK_VM_INFO_COUNT;
    kern_return_t kernelReturn = task_info(mach_task_self(), TASK_VM_INFO, (task_info_t) &vmInfo, &count);
    if(kernelReturn == KERN_SUCCESS) {
        memoryUsageInByte = (int64_t) vmInfo.phys_footprint;
        NSLog(@"Memory in use (in bytes): %lld", memoryUsageInByte);
    } else {
        NSLog(@"Error with task_info(): %s", mach_error_string(kernelReturn));
    }

    return memoryUsageInByte;
}

```

```
[NSProcessInfo processInfo].physicalMemory
```

