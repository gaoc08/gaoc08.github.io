---
layout:     post
title:      iOS Runtime 原理
subtitle:   
date:       2019-12-26
author:     G
header-img: img/post-bg-ioses.jpg
catalog: true
tags:
    - iOS
    - Runtime
    - Category
    - Method Swizzling
    - 深入原理系列
---



# 0. 概述



Runtime 是 OC 区别于静态语言的一个非常重要的特性。

对于静态语言，函数的调用会在编译期就已经决定好，在编译完成后直接顺序执行。对于 OC 这门动态语言，函数调用变成了消息发送，在编译期不能知道要调用哪个函数。

Runtime 就解决了运行时如何正确处理消息发送的问题。





# 1. 类和 NSObject 基本数据结构

### 1. 实例 & 类 & 元类

##### 1. 实例(objc_object)

```c
/// Represents an instance of a class.
struct objc_object {
    Class isa  OBJC_ISA_AVAILABILITY;
};

/// A pointer to an instance of a class.
typedef struct objc_object *id;
```

##### 2. 类(objc_class)

```c
typedef struct objc_class *Class;
struct objc_class {
	Class isa OBJC_ISA_AVAILABILITY; //isa指针指向Meta Class，因为Objc的类的本身也是一个Object，为了处理这个关系，runtime就创造了Meta Class，当给类发送[NSObject alloc]这样消息时，实际上是把这个消息发给了Class Object

#if !__OBJC2__
	Class super_class; // 父类
	const char *name; // 类名
	long version; // 类的版本信息，默认为0
	long info; // 类信息，供运行期使用的一些位标识
	long instance_size; // 该类的实例变量大小
	struct objc_ivar_list *ivars; // 该类的成员变量链表
	struct objc_method_list **methodLists; // 方法定义的链表
	struct objc_cache *cache; // 方法缓存，对象接到一个消息会根据isa指针查找消息对象，这时会在methodLists中遍历，如果cache了，常用的方法调用时就能够提高调用的效率。
	struct objc_protocol_list *protocols; // 协议链表
#endif
} OBJC2_UNAVAILABLE;

```

##### 3.元类(Meta class)

> 元类是一个概念，并不是一个新的 struct。



##### 4. 实例、类与元类直接的关系

> 引用一张经典的 实例 - 类 - 元类 的关系图。

<img src="https://tva1.sinaimg.cn/large/006tNbRwly1gaazvpk1j7j30px0r5gmy.jpg" alt="16947663bd7ddd2" style="zoom:50%;" />

1. 实例是类的实例化，实例中包含着 isa 指针，指向它实例化的类。
2. 类是元类的实例化，类也是一个对象，并且每个类对象都是单例。类的 struct 中包含了 isa 指针，指向它的原类。
3. 类中保存着实例方法列表；元类中包含了所有类方法的列表。
4. 类是普通实例的描述，元类是类（对象）的描述。
5. 元类的方法列表是类方法（类响应的选择器）。当你发送消息给一个类（元类的实例），objc_msgSend() 会查找元类（已及它的父类）的方法列表并确定调用的方法。
6. 对于 isa 链，最终的结果是： 实例 -> 类 -> 元类 -> 根类的原类 -> 根类的原类自己。
7. 我们首先看类的父类链： 子类 -> 父类 -> 根类 -> nil。
8. 我们再看元类的父类链：子元类 -> 父元类 -> 根元类 -> 根类 -> nil。
9. 通过以上两条父类链我们可以看到，元类的父类链和类的父类链是平行的，所以类方法和实例方法一样得到继承。

### 2. Method & SEL & IMP

##### 1. 方法(objc_method)

```c
typedef struct objc_method *Method;

struct objc_method {
    SEL method_name; //方法名
    char *method_types; //方法类型
    IMP method_imp; //方法实现
} OBJC2_UNAVAILABLE;
```



##### 2. SEL (objc_selector)

```c
/// An opaque type that represents a method selector.
typedef struct objc_selector *SEL;
```



##### 3. IMP

```c
/// A pointer to the function of a method implementation. 
#if !OBJC_OLD_DISPATCH_PROTOTYPES
typedef void (*IMP)(void /* id, SEL, ... */ ); 
#else
typedef id (*IMP)(id, SEL, ...); 
#endif
```

> IMP 可以理解为函数指针，指向了函数的实现代码。这个被指向的函数的参数包含一个接收消息的 id 对象、 SEL、以及不定个数的参数，并返回一个 id 类型的对象，我们可以像在Ｃ语言里面一样使用这个函数指针。



##### 4. 总结

由于 SEL 只和函数名字有关，所以导致 iOS 不支持函数的重载（函数名相同，参数不同），不过 iOS 支持函数的重写，因为不同类可以有同名的 SEL。

IMP 和 SEL 在在 method 中是类似 key-value 的存储方式。所以 runtime 根据 SEL 找到相应的 IMP 来完成方法的调用。



### 3.缓存(objc_cache)

1. ##### 为何要有缓存

   - 从上面的介绍中我们可以发现，方法在类中的存储是通过 list 来存储的。所以方法的查找就需要遍历 method_list，然后通过 SEL 的比对来找到 相应方法的 IMP，最终完成函数的调用。

   - 但是如果某个方法要进行高频次的调用，比如

     ```objective-c
     MyClass *mc = [MyClass new];
     for (int i = 0; i < 10000; i++) {
       [mc myMethod];
     }
     ```

   - 特别是如果 myMethod 出现在 MyClass 的根类中，然后 MyClass 和父类的方法列表都很大的时候。那性能的消耗就难以想象了。。。

2. ##### 定义

   ```c
   typedef struct objc_cache *Cache
   struct objc_cache {
       unsigned int mask /* total = mask + 1 */; // 当前能达到的最大 index
       unsigned int occupied; // 被占用的槽位，因为缓存是以散列表的形式存在的，所以会有空槽，而occupied表示当前被占用的数目
       Method buckets[1]; //用数组表示的 hash 表，Method 类型，每一个 Method 代表一个方法缓存，buckets 是可变数组。
   };
   ```

3. ##### 如何实现缓存

   存储的数据结构使用了散列表。

   

### 4. Category(objc_category)



# 2. Runtime 整体流程概述



# 3. 消息传递机制



# 4. 消息转发机制



# 5. Runtime 的应用



# 6. Runtime 的相关问题

