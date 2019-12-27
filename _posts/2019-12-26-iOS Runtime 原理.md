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



## 0. 概述



Runtime 是 OC 区别于静态语言的一个非常重要的特性。

对于静态语言，函数的调用会在编译期就已经决定好，在编译完成后直接顺序执行。对于 OC 这门动态语言，函数调用变成了消息发送，在编译期不能知道要调用哪个函数。

Runtime 就解决了运行时如何正确处理消息发送的问题。





## 1. 类和 NSObject 基本数据结构

#### 1. 类(objc_class)

```c
typedef struct objc_class *Class;
struct objc_class {
	Class isa OBJC_ISA_AVAILABILITY; //isa指针指向Meta Class，因为Objc的类的本身也是一个Object，为了处理这个关系，runtime就创造了Meta Class，当给类发送[NSObject alloc]这样消息时，实际上是把这个消息发给了Class Object

#if !__OBJC2__
	Class super_class OBJC2_UNAVAILABLE; // 父类
	const char *name OBJC2_UNAVAILABLE; // 类名
	long version OBJC2_UNAVAILABLE; // 类的版本信息，默认为0
	long info OBJC2_UNAVAILABLE; // 类信息，供运行期使用的一些位标识
	long instance_size OBJC2_UNAVAILABLE; // 该类的实例变量大小
	struct objc_ivar_list *ivars OBJC2_UNAVAILABLE; // 该类的成员变量链表
	struct objc_method_list **methodLists OBJC2_UNAVAILABLE; // 方法定义的链表
	struct objc_cache *cache OBJC2_UNAVAILABLE; // 方法缓存，对象接到一个消息会根据isa指针查找消息对象，这时会在methodLists中遍历，如果cache了，常用的方法调用时就能够提高调用的效率。
	struct objc_protocol_list *protocols OBJC2_UNAVAILABLE; // 协议链表
#endif
} OBJC2_UNAVAILABLE;

```



#### 2. 实例(objc_object)

```c
/// Represents an instance of a class.
struct objc_object {
    Class isa  OBJC_ISA_AVAILABILITY;
};

/// A pointer to an instance of a class.
typedef struct objc_object *id;
```



#### 3.元类(Meta class)

> 元类是一个概念，并不是一个新的 struct。

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

#### 4. 方法(objc_method)



#### 5. SEL (objc_selector)



#### 6. IMP



#### 7. Category(objc_category)



#### 8. 类缓存(objc_cache)



## 2. Runtime 整体流程概述



## 3. 消息传递机制



## 4. 消息转发机制



## 5. Runtime 的应用



## 6. Runtime 的相关问题

