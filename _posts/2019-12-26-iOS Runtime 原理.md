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

##### 1. 为何要有缓存

- 从上面的介绍中我们可以发现，方法在类中的存储是通过 list 来存储的。所以方法的查找就需要遍历 method_list，然后通过 SEL 的比对来找到 相应方法的 IMP，最终完成函数的调用。

- 但是如果某个方法要进行高频次的调用，比如

  ```objective-c
  MyClass *mc = [MyClass new];
  for (int i = 0; i < 10000; i++) {
    [mc myMethod];
  }
  ```

- 特别是如果 myMethod 出现在 MyClass 的根类中，然后 MyClass 和父类的方法列表都很大的时候。那性能的消耗就难以想象了。。。

##### 2. 定义

```c
typedef struct objc_cache *Cache
struct objc_cache {
    unsigned int mask /* total = mask + 1 */; // 当前能达到的最大 index
    unsigned int occupied; // 被占用的槽位，因为缓存是以散列表的形式存在的，所以会有空槽，而occupied表示当前被占用的数目
    Method buckets[1]; //用数组表示的 hash 表，Method 类型，每一个 Method 代表一个方法缓存，buckets 是可变数组。
};
```

##### 3. 如何实现缓存

存储的数据结构使用了散列表。

1. cache 的写入：

   ```c
   static void cache_fill_nolock(Class cls, SEL sel, IMP imp, id receiver)
   {
       cacheUpdateLock.assertLocked();
   
       // Never cache before +initialize is done
       if (!cls->isInitialized()) return;
   
       // Make sure the entry wasn't added to the cache by some other thread 
       // before we grabbed the cacheUpdateLock.
       if (cache_getImp(cls, sel)) return;
   
       cache_t *cache = getCache(cls);
       cache_key_t key = getKey(sel);
   
       // Use the cache as-is if it is less than 3/4 full
       mask_t newOccupied = cache->occupied() + 1;
       mask_t capacity = cache->capacity();
       if (cache->isConstantEmptyCache()) {
           // Cache is read-only. Replace it.
           cache->reallocate(capacity, capacity ?: INIT_CACHE_SIZE);
       }
       else if (newOccupied <= capacity / 4 * 3) {
           // Cache is less than 3/4 full. Use it as-is.
       }
       else {
           // Cache is too full. Expand it.
           cache->expand();
       }
   
       // Scan for the first unused slot and insert there.
       // There is guaranteed to be an empty slot because the 
       // minimum size is 4 and we resized at 3/4 full.
       bucket_t *bucket = cache->find(key, receiver);
       if (bucket->key() == 0) cache->incrementOccupied();
       bucket->set(key, imp);
   }
   
   bucket_t * cache_t::find(cache_key_t k, id receiver)
   {
       assert(k != 0);
   
       bucket_t *b = buckets();
       mask_t m = mask();
       mask_t begin = cache_hash(k, m);
       mask_t i = begin;
       do {
           if (b[i].key() == 0  ||  b[i].key() == k) {
               return &b[i];
           }
       } while ((i = cache_next(i, m)) != begin);
   
       // hack
       Class cls = (Class)((uintptr_t)this - offsetof(objc_class, cache));
       cache_t::bad_cache(receiver, (SEL)k, cls);
   }
   
   static inline mask_t cache_next(mask_t i, mask_t mask) {
       return (i+1) & mask;
   }
   
   ```

   上面的注释已经写的很清楚了。简单总结一下：

   - 如果 cache 为还未创建，那么创建一个 INIT_CACHE_SIZE 的 cache。
   - 如果使用量小于四分之三，那么直接向散列表中添加条目。
   - 如果使用量超过了四分之三，那么 cache 扩容，扩容为之前容量的二倍。如果溢出 mask 最大值则不扩容。
   - 通过 fill 中的 find 函数，我们可以发现，这里对于哈希碰撞是采用了开放地址法解决的，也就是从碰撞点开始向后查找，找到一个空 bucket 作为 fill 的目标桶。

2. cache 的读：

   ```c
   extern IMP cache_getImp(Class cls, SEL sel);
   ```

   runtime 代码中没有找到 cache_getImp 的实现，网上查资料看到[一篇掘金的文章](https://juejin.im/post/5db30a7af265da4d57770f2b#heading-1)描写的很详细，可以作为参考。

### 4. Category(objc_category)

##### 1. 定义

```c
/// An opaque type that represents a category.
typedef struct objc_category *Category;

struct objc_category {
    char *category_name; // category 的名字
    char *class_name; // 类的名字
    struct objc_method_list *instance_methods; // 实例方法列表
    struct objc_method_list *class_methods; // 类方法列表
    struct objc_protocol_list *protocols; // 协议列表
};

struct category_t {
    const char *name; // 要扩展的类的名字，非 category 名字
    classref_t cls; // 要扩展的类的引用
    struct method_list_t *instanceMethods; // 实例方法列表
    struct method_list_t *classMethods; // 类方法列表
    struct protocol_list_t *protocols; // 协议列表
    struct property_list_t *instanceProperties; // 实例属性列表

    method_list_t *methodsForMeta(bool isMeta) {
        if (isMeta) return classMethods; // 元类返回的是类方法列表
        else return instanceMethods; // 非元类返回的是实例方法列表
    }

    property_list_t *propertiesForMeta(bool isMeta) {
        if (isMeta) return nil; // classProperties;
        else return instanceProperties;
    }
};
```

- Category 中可以添加实例方法、类方法、协议。
- Category 中有属性列表。
- Category 中没有实例变量列表，所以无法自动生成属性的 get/set 方法，这也是我们无法直接在 category中 添加属性的原因。但是可以通过关联对象添加属性。

##### 2. 实现

```c
/***********************************************************************
* _read_images
* Perform initial processing of the headers in the linked 
* list beginning with headerList. 
*
* Called by: map_images_nolock
*
* Locking: runtimeLock acquired by map_images
**********************************************************************/
void _read_images(header_info **hList, uint32_t hCount)
{
		// 省略代码。。。
		// Discover categories. 
    for (EACH_HEADER) {
        category_t **catlist = 
            _getObjc2CategoryList(hi, &count);
        for (i = 0; i < count; i++) {
            category_t *cat = catlist[i];
            Class cls = remapClass(cat->cls);

            if (!cls) {
                // Category's target class is missing (probably weak-linked).
                // Disavow any knowledge of this category.
                catlist[i] = nil;
                if (PrintConnecting) {
                    _objc_inform("CLASS: IGNORING category \?\?\?(%s) %p with "
                                 "missing weak-linked target class", 
                                 cat->name, cat);
                }
                continue;
            }

            // Process this category. 
            // First, register the category with its target class. 
            // Then, rebuild the class's method lists (etc) if 
            // the class is realized. 
            bool classExists = NO;
            if (cat->instanceMethods ||  cat->protocols  
                ||  cat->instanceProperties) 
            {
                addUnattachedCategoryForClass(cat, cls, hi);
                if (cls->isRealized()) {
                    remethodizeClass(cls);
                    classExists = YES;
                }
                if (PrintConnecting) {
                    _objc_inform("CLASS: found category -%s(%s) %s", 
                                 cls->nameForLogging(), cat->name, 
                                 classExists ? "on existing class" : "");
                }
            }

            if (cat->classMethods  ||  cat->protocols  
                /* ||  cat->classProperties */) 
            {
                addUnattachedCategoryForClass(cat, cls->ISA(), hi);
                if (cls->ISA()->isRealized()) {
                    remethodizeClass(cls->ISA());
                }
                if (PrintConnecting) {
                    _objc_inform("CLASS: found category +%s(%s)", 
                                 cls->nameForLogging(), cat->name);
                }
            }
        }
    }

    ts.log("IMAGE TIMES: discover categories");

    // Category discovery MUST BE LAST to avoid potential races 
    // when other threads call the new category code before 
    // this thread finishes its fixups.

		// 省略代码。。。
}


/***********************************************************************
* remethodizeClass
* Attach outstanding categories to an existing class.
* Fixes up cls's method list, protocol list, and property list.
* Updates method caches for cls and its subclasses.
* Locking: runtimeLock must be held by the caller
**********************************************************************/
static void remethodizeClass(Class cls)
{
    category_list *cats;
    bool isMeta;

    runtimeLock.assertWriting();

    isMeta = cls->isMetaClass();

    // Re-methodizing: check for more categories
    if ((cats = unattachedCategoriesForClass(cls, false/*not realizing*/))) {
        if (PrintConnecting) {
            _objc_inform("CLASS: attaching categories to class '%s' %s", 
                         cls->nameForLogging(), isMeta ? "(meta)" : "");
        }
        
        attachCategories(cls, cats, true /*flush caches*/);        
        free(cats);
    }
}

// Attach method lists and properties and protocols from categories to a class.
// Assumes the categories in cats are all loaded and sorted by load order, 
// oldest categories first.
static void 
attachCategories(Class cls, category_list *cats, bool flush_caches)
{
    if (!cats) return;
    if (PrintReplacedMethods) printReplacements(cls, cats);

    bool isMeta = cls->isMetaClass();

    // fixme rearrange to remove these intermediate allocations
    method_list_t **mlists = (method_list_t **)
        malloc(cats->count * sizeof(*mlists));
    property_list_t **proplists = (property_list_t **)
        malloc(cats->count * sizeof(*proplists));
    protocol_list_t **protolists = (protocol_list_t **)
        malloc(cats->count * sizeof(*protolists));

    // Count backwards through cats to get newest categories first
    int mcount = 0;
    int propcount = 0;
    int protocount = 0;
    int i = cats->count;
    bool fromBundle = NO;
    while (i--) {
        auto& entry = cats->list[i];

        method_list_t *mlist = entry.cat->methodsForMeta(isMeta);
        if (mlist) {
            mlists[mcount++] = mlist;
            fromBundle |= entry.hi->isBundle();
        }

        property_list_t *proplist = entry.cat->propertiesForMeta(isMeta);
        if (proplist) {
            proplists[propcount++] = proplist;
        }

        protocol_list_t *protolist = entry.cat->protocols;
        if (protolist) {
            protolists[protocount++] = protolist;
        }
    }

    auto rw = cls->data();

    prepareMethodLists(cls, mlists, mcount, NO, fromBundle);
    rw->methods.attachLists(mlists, mcount);
    free(mlists);
    if (flush_caches  &&  mcount > 0) flushCaches(cls);

    rw->properties.attachLists(proplists, propcount);
    free(proplists);

    rw->protocols.attachLists(protolists, protocount);
    free(protolists);
}

    void attachLists(List* const * addedLists, uint32_t addedCount) {
        if (addedCount == 0) return;

        if (hasArray()) {
            // many lists -> many lists
            uint32_t oldCount = array()->count;
            uint32_t newCount = oldCount + addedCount;
            setArray((array_t *)realloc(array(), array_t::byteSize(newCount)));
            array()->count = newCount;
            memmove(array()->lists + addedCount, array()->lists, 
                    oldCount * sizeof(array()->lists[0]));
            memcpy(array()->lists, addedLists, 
                   addedCount * sizeof(array()->lists[0]));
        }
        else if (!list  &&  addedCount == 1) {
            // 0 lists -> 1 list
            list = addedLists[0];
        } 
        else {
            // 1 list -> many lists
            List* oldList = list;
            uint32_t oldCount = oldList ? 1 : 0;
            uint32_t newCount = oldCount + addedCount;
            setArray((array_t *)malloc(array_t::byteSize(newCount)));
            array()->count = newCount;
            if (oldList) array()->lists[addedCount] = oldList;
            memcpy(array()->lists, addedLists, 
                   addedCount * sizeof(array()->lists[0]));
        }
    }

```

- `_read_images` 中的部分代码给出了 Category 的整体加载流程。先看实例相关的，再看类相关的。都是先把 cat 加到 unattached 列表中，然后通过 `remethodizeClass` 方法，将 unattached 的 cats 通过 `attachCategories` 方法 attach 上。
- 通过` _read_images` 中代码可以发现 protocol 是在类和元类中都存储的。
- 通过 `attachLists` 函数我们可以发现，Category 的方法等列表都是插入到被扩展的 Class 的列表的前面。
- 对于函数的方法查找来说，都是通过 list 遍历，这样就导致了 Category 的方法实现可以覆盖被扩展的类。对于同一个类来说，后编译的 Category 的方法会覆盖先编译的 Category。

##### 3. 关联对象

> 使用：
>
> ```objective-c
> -(void)setName:(NSString *)name
> {
>     objc_setAssociatedObject(self, @"name",name, OBJC_ASSOCIATION_RETAIN_NONATOMIC);
> }
> -(NSString *)name
> {
>     return objc_getAssociatedObject(self, @"name");    
> }
> ```



1. 

# 2. Runtime 整体流程概述



# 3. 消息传递机制



# 4. 消息转发机制



# 5. Runtime 的应用



# 6. Runtime 的相关问题

