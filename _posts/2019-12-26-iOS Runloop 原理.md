---
layout:     post
title:      iOS Runloop 原理
subtitle:   
date:       2019-12-26
author:     G
header-img: img/post-bg-ioses.jpg
catalog: true
tags:
    - iOS
    - Runloop
    - 深入原理系列
---



# 1. Runloop 基本概念

> Runloop 是和线程相关的基础设置的一部分。Runloop 是用来规划和协调工作和到来事件的一个事件循环。目的就是让线程在有事情做的时候忙碌起来，在没事可做的时候休眠。

> CFRunLoopRef 是在 CoreFoundation 框架内的，它提供了纯 C 函数的 API，所有这些 API 都是线程安全的。NSRunLoop 是基于 CFRunLoopRef 的封装，提供了面向对象的 API，但是这些 API 不是线程安全的。

### 1. 整体结构

##### 1. 源码

```c
struct __CFRunLoop {
    CFRuntimeBase _base;
    _CFRecursiveMutex _lock;			/* locked for accessing mode list */
    __CFPort _wakeUpPort;			// used for CFRunLoopWakeUp 
    Boolean _unused;
    volatile _per_run_data *_perRunData;              // reset for runs of the run loop
    _CFThreadRef _pthread;
    uint32_t _winthread;
    CFMutableSetRef _commonModes;
    CFMutableSetRef _commonModeItems;
    CFRunLoopModeRef _currentMode;
    CFMutableSetRef _modes;
    struct _block_item *_blocks_head;
    struct _block_item *_blocks_tail;
    CFAbsoluteTime _runTime;
    CFAbsoluteTime _sleepTime;
    CFTypeRef _counterpart;
    _Atomic(uint8_t) _fromTSD;
    CFLock_t _timerTSRLock;
};

```

##### 2. 图示

<img src="https://tva1.sinaimg.cn/large/006tNbRwly1gakaggjzj3j30og0xodj7.jpg" alt="162a5b83211b0c82" style="zoom:80%;" />

##### 3. 详细细节

我们通过源码和上图可以发现：

1. Runloop 中有一个线程、一个 CommonModes 的集合、一个 commonModeItems 的集合、一个所有 mode 的集合、以及一个当前 mode。
2. Mode 中包含一个名字、一个 source0 的集合、一个 source1 的集合、一个 observer 的集合 和一个 timers 的集合。
3. 整体上就是 Runloop 包含很多 mode，每个 mode 管理自己的 source、observer、timer 等信息，而 Runloop 通过 currentMode 管理所处的 mode 状态。

### 2. Runloop Mode

##### 1. 源码

```objective-c
typedef struct __CFRunLoopMode *CFRunLoopModeRef;

struct __CFRunLoopMode {
    CFRuntimeBase _base;
    _CFRecursiveMutex _lock;	/* must have the run loop locked before locking this */
    CFStringRef _name;
    Boolean _stopped;
    char _padding[3];
    CFMutableSetRef _sources0; //
    CFMutableSetRef _sources1;
    CFMutableArrayRef _observers;
    CFMutableArrayRef _timers;
    CFMutableDictionaryRef _portToV1SourceMap;
    __CFPortSet _portSet;
    CFIndex _observerMask;
#if USE_DISPATCH_SOURCE_FOR_TIMERS
    dispatch_source_t _timerSource;
    dispatch_queue_t _queue;
    Boolean _timerFired; // set to true by the source when a timer has fired
    Boolean _dispatchTimerArmed;
#endif
#if USE_MK_TIMER_TOO
    __CFPort _timerPort;
    Boolean _mkTimerArmed;
#endif
#if TARGET_OS_WIN32
    DWORD _msgQMask;
    void (*_msgPump)(void);
#endif
    uint64_t _timerSoftDeadline; /* TSR */
    uint64_t _timerHardDeadline; /* TSR */
};
```

##### 2. 详细细节

1. Runloop 管理 mode，mode 管理 source、observer、timer、port 等事件。
2. Runloop 某一时刻只能跑在一种 mode 下，也就是 currentMode。此时 runloop 只能处理 currentMode 中的 source、timer、observer 等。
3. Runloop 要想切换 mode 只能退出一个 mode 再进入另一个 mode，这样保证了 mode 中处理的事件互相独立。
4. 苹果文档中提到的 Mode 有五个，分别是：
   - NSDefaultRunLoopMode
   - NSConnectionReplyMode
   - NSModalPanelRunLoopMode
   - NSEventTrackingRunLoopMode
   - NSRunLoopCommonModes
5. 我们一般接触到的是 NSDefaultRunLoopMode。
6. NSRunLoopCommonModes 是一个 mode 的集合，默认包含 NSDefaultRunLoopMode 和 NSEventTrackingRunLoopMode 两个 mode。
7. 可以创建自定义的 mode ，也可以将其他 mode 添加到 NSRunLoopCommonModes 中。

##### 3. 相关 api

1. CFRunLoopAddCommonMode(CFRunLoopRef rl, CFStringRef mode)

   > 向当前 RunLoop 的 commonModes 中添加一个 mode。

   1. 源码

      ```objective-c
      void CFRunLoopAddCommonMode(CFRunLoopRef rl, CFStringRef modeName) {
          CHECK_FOR_FORK();
          if (__CFRunLoopIsDeallocating(rl)) return;
          __CFRunLoopLock(rl);
          if (!CFSetContainsValue(rl->_commonModes, modeName)) {
      	CFSetRef set = rl->_commonModeItems ? CFSetCreateCopy(kCFAllocatorSystemDefault, rl->_commonModeItems) : NULL;
      	CFSetAddValue(rl->_commonModes, modeName);
      	if (NULL != set) {
      	    CFTypeRef context[2] = {rl, modeName};
      	    /* add all common-modes items to new mode */
      	    CFSetApplyFunction(set, (__CFRunLoopAddItemsToCommonMode), (void *)context);
      	    CFRelease(set);
      	}
          } else {
          }
          __CFRunLoopUnlock(rl);
      }
      ```

   2. 结论

      - ModeName 是 Mode 的唯一标识符，不能重复。
      - 如果 mode 不存在，那么 mode 会被创建。
      - 对于一个 RunLoop 来说，其内部的 mode 只能增加不能删除。
      - 添加 commonMode 会把 commonModeItems 数组中的所有 source 同步到新添加的 mode 中。

2. CFStringRef CFRunLoopCopyCurrentMode(CFRunLoopRef rl)

   > 返回当前运行的 mode 的 name

3. CFArrayRef CFRunLoopCopyAllModes(CFRunLoopRef rl)

   > 返回当前 RunLoop 的 所有 mode 的 name

### 3. Runloop Source

##### 1. 概览

> Runloop 接收两种不同类型的 source，第一种是 Input Source，第二种是 Timer source。

![runloop](https://tva1.sinaimg.cn/large/006tNbRwly1ganqscx1avj30dg071wfo.jpg)

##### 2. Input Source

1. Input Source 传递异步事件，通常是从其他线程或者其他应用来的消息。
2. Input Source 通常分为两大类：Port-based Input Source 和 Custom Input Source。
3. Port-based 监控你应用的 mach port。Custom 监控自定义 source 的事件。
4. 两种 Input source 最大的区别在于如何触发：Port-based 是内核自动触发，Custom 是需要手动触发的。
5. 对于 __CFRunLoopSource 来说，Port-based 就是 version1，Custom 的就是 version0。
6. 对于 __CFRunLoopMode 来说，source0 是 Custom 的集合，source1 是 Port-based 的集合。
7. 5 和 6 的结论可以从 [苹果的如何配置 Port-based 和 Custom Input Source 的文档](https://developer.apple.com/library/archive/documentation/Cocoa/Conceptual/Multithreading/RunLoopManagement/RunLoopManagement.html#//apple_ref/doc/uid/10000057i-CH16-131281) 中得出。
8. 还有一种比较特殊的是 Cocoa 的 Perform Selector Sources。
   - 在执行完之后就会自动从 runloop 中移除。
   - 需要有 active 的 runloop 才能在该线程上 perform selector。
   - runloop 会在一个 loop 中执行完所有在排队的 perform selector source，而不是一个 loop 处理一个 source。

##### 3. Timer Source

1. Timer Source 传递同步事件，通常发生在特定事件，或者每过重复间隔时间重复发生。

2. Timer 并不是一个 real-time 的机制。

3. Timer 是需要跑在一个 Runloop Mode 下的，如果 Timer 所在的 mode 并没有运行，那么 Timer 会一直等到所在的 Mode 运行的时候才会触发。

4. 在实验中，向 runloop defaultMode 添加了 10 个 timer 分别是 1-10 秒后执行，然后此时不停滑动 scrollview 15 秒，停止滑动之后，10 个 timer 依次在很短时间内触发。注意，timer 没有丢失。

   <img src="https://tva1.sinaimg.cn/large/006tNbRwly1ganzof58jfj30qo0a8go0.jpg" alt="image-20200107143821746" style="zoom:50%;" />

5. 在第二个实验中，向 runloop defaultMode 中，添加一个 1 秒重复的 timer。然后中间滑动 ScrollView。

   1. 我们通过后两个红圈可以看到：当 runloop default mode 重新执行的时候，不管之前错过了多少个 timer 触发时机，都只会立刻触发 **一次**，触发一次之后恢复最开始的 timer loop。
   2. 特别的，我们通过第一个红圈处可以看到：当 default mode 恰好在 timer 触发时间范围内重新开始的时候，timer 会触发一次，然后依旧 1 秒一次。

   <img src="https://tva1.sinaimg.cn/large/006tNbRwly1gapebtkxadj30rm0f078h.jpg" alt="截屏2020-01-0714.42.42" style="zoom:50%;" />

6. CFRunLoopTimer 和 NSTimer 是 [Toll-Free Bridged](https://developer.apple.com/library/archive/documentation/CoreFoundation/Conceptual/CFDesignConcepts/Articles/tollFreeBridgedTypes.html#//apple_ref/doc/uid/TP40010677)，也就是可以互相转换。

### 4. Runloop Observer

1. CFRunLoopObserver 观察 Runloop 的各种状态，并且提供回调在相应的时机处理事情。

2. 目前 runloop 的 observer 主要有以下几种，含义从名字也可以看的很清楚。

   ```c
   /* Run Loop Observer Activities */
   typedef CF_OPTIONS(CFOptionFlags, CFRunLoopActivity) {
       kCFRunLoopEntry = (1UL << 0),
       kCFRunLoopBeforeTimers = (1UL << 1),
       kCFRunLoopBeforeSources = (1UL << 2),
       kCFRunLoopBeforeWaiting = (1UL << 5),
       kCFRunLoopAfterWaiting = (1UL << 6),
       kCFRunLoopExit = (1UL << 7),
       kCFRunLoopAllActivities = 0x0FFFFFFFU
   };
   ```

### 5. Runloop 与线程

##### 1. 先看代码

```objective-c
CFRunLoopRef CFRunLoopGetMain(void) {
    CHECK_FOR_FORK();
    static CFRunLoopRef __main = NULL; // no retain needed
    if (!__main) __main = _CFRunLoopGet0(pthread_main_thread_np()); // no CAS needed
    return __main;
}

CFRunLoopRef CFRunLoopGetCurrent(void) {
    CHECK_FOR_FORK();
    CFRunLoopRef rl = (CFRunLoopRef)_CFGetTSD(__CFTSDKeyRunLoop);
    if (rl) return rl;
    return _CFRunLoopGet0(pthread_self());
}

// should only be called by Foundation
// t==0 is a synonym for "main thread" that always works
CF_EXPORT CFRunLoopRef _CFRunLoopGet0(_CFThreadRef t) {
    if (pthread_equal(t, kNilPthreadT)) {
        t = pthread_main_thread_np();
    }
    __CFLock(&loopsLock);
    if (!__CFRunLoops) { // 如果还没有全局的__CFRunLoops，那么创建并且创建主线程的 runloop 并加到其中
        CFMutableDictionaryRef dict = CFDictionaryCreateMutable(kCFAllocatorSystemDefault, 0, NULL, &kCFTypeDictionaryValueCallBacks);
        CFRunLoopRef mainLoop = __CFRunLoopCreate(pthread_main_thread_np());
        CFDictionarySetValue(dict, pthreadPointer(pthread_main_thread_np()), mainLoop);
        if (!OSAtomicCompareAndSwapPtrBarrier(NULL, dict, (void * volatile *)&__CFRunLoops)) {
            CFRelease(dict);
        }
        CFRelease(mainLoop);
    }
    CFRunLoopRef newLoop = NULL;
  	// 从全局的 __CFRunLoops 中获取线程所对应的 runloop
    CFRunLoopRef loop = (CFRunLoopRef)CFDictionaryGetValue(__CFRunLoops, pthreadPointer(t));
  	// 如果该线程没有对应的 runloop，那么就创建一个并加入 __CFRunLoops 中。
    if (!loop) {
        newLoop = __CFRunLoopCreate(t);
        CFDictionarySetValue(__CFRunLoops, pthreadPointer(t), newLoop);
        loop = newLoop;
    }
    __CFUnlock(&loopsLock);
    // don't release run loops inside the loopsLock, because CFRunLoopDeallocate may end up taking it
    if (newLoop) { CFRelease(newLoop); }
    
    if (pthread_equal(t, pthread_self())) {
        _CFSetTSD(__CFTSDKeyRunLoop, (void *)loop, NULL);
        if (0 == _CFGetTSD(__CFTSDKeyRunLoopCntr)) {
#if _POSIX_THREADS
        _CFSetTSD(__CFTSDKeyRunLoopCntr, (void *)(PTHREAD_DESTRUCTOR_ITERATIONS-1), (void (*)(void *))__CFFinalizeRunLoop);
#else
        _CFSetTSD(__CFTSDKeyRunLoopCntr, 0, &__CFFinalizeRunLoop);
#endif
        }
    }
    return loop;
}
```

##### 2. 结论和解析

1. 我们不能主动创建 runloop，CF 提供了 CFRunLoopGetMain() 和 CFRunLoopGetCurrent() 两个方法来获取 runloop。
2. 两个获取 runloop 的方法都是调用了 `CFRunLoopRef _CFRunLoopGet0(_CFThreadRef t)` 方法。
3. runloop 和 thread 是一一对应的关系，并且存在一个 `__CFRunLoops` 的全局可变字典中。
4. 子线程创建时，并不会自动创建 runloop。主线程会在程序启动的时候自动创建 runloop。
5. 对于主线程，会在 `__CFRunLoops` 字典创建的时候就创建好 runloop 并且添加到 `__CFRunLoops` 中。
6. 对于子线程，会在第一次获取 runloop 的时候，创建对应的 runloop 并加到 `__CFRunLoops` 中。

# 2. Runloop 流程

流程图基于[苹果文档](https://developer.apple.com/library/archive/documentation/Cocoa/Conceptual/Multithreading/RunLoopManagement/RunLoopManagement.html#//apple_ref/doc/uid/10000057i-CH16-131281)的 The Run Loop Sequence of Events 部分。并且参考 Foundation 的源码。

![Runloop 流程](https://tva1.sinaimg.cn/large/006tNbRwly1gaq3xd5eqfj30u01dxwld.jpg)



# 3. Runloop 使用



# 4. Runloop 应用



### 1. AutoReleasePool





### 2. Timer



### 3. 事件响应



### 4. 手势识别



### 5. 界面更新



### 6. 常驻线程









# 参考文章

1. [掘金 - iOS RunLoop详解](https://juejin.im/post/5aca2b0a6fb9a028d700e1f8)
2. [深入理解RunLoop](https://blog.ibireme.com/2015/05/18/runloop/)


