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
struct objc_class : objc_object {
    // Class ISA;
    Class superclass;
    cache_t cache;             // formerly cache pointer and vtable
    class_data_bits_t bits;    // class_rw_t * plus custom rr/alloc flags
    class_rw_t *data() { 
        return bits.data();
    }
    // 省略一些从 bits 中获取信息的方法
}

struct class_rw_t {
    uint32_t flags;
    uint32_t version;

    const class_ro_t *ro;

    method_array_t methods;
    property_array_t properties;
    protocol_array_t protocols;

    Class firstSubclass;
    Class nextSiblingClass;

    char *demangledName;
    // 省略部分方法
};

struct class_ro_t {
    uint32_t flags;
    uint32_t instanceStart;
    uint32_t instanceSize;
#ifdef __LP64__
    uint32_t reserved;
#endif

    const uint8_t * ivarLayout;
    
    const char * name;
    method_list_t * baseMethodList;
    protocol_list_t * baseProtocols;
    const ivar_list_t * ivars;

    const uint8_t * weakIvarLayout;
    property_list_t *baseProperties;

    method_list_t *baseMethods() const {
        return baseMethodList;
    }
};

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

##### 1. 方法(method_t)

```c
typedef struct method_t *Method;
struct method_t {
    SEL name;
    const char *types;
    IMP imp;

    struct SortBySELAddress :
        public std::binary_function<const method_t&,
                                    const method_t&, bool>
    {
        bool operator() (const method_t& lhs,
                         const method_t& rhs)
        { return lhs.name < rhs.name; }
    };
};
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



### 3.缓存(cache_t)

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
struct cache_t {
    struct bucket_t *_buckets; //用数组表示的 hash 表，Method 类型，每一个 Method 代表一个方法缓存，buckets 是可变数组。
    mask_t _mask; // 当前能达到的最大 index
    mask_t _occupied; // 被占用的槽位，因为缓存是以散列表的形式存在的，所以会有空槽，而occupied表示当前被占用的数目

public:
    struct bucket_t *buckets();
    mask_t mask();
    mask_t occupied();
    void incrementOccupied();
    void setBucketsAndMask(struct bucket_t *newBuckets, mask_t newMask);
    void initializeToEmpty();

    mask_t capacity();
    bool isConstantEmptyCache();
    bool canBeFreed();

    static size_t bytesForCapacity(uint32_t cap);
    static struct bucket_t * endMarker(struct bucket_t *b, uint32_t cap);

    void expand();
    void reallocate(mask_t oldCapacity, mask_t newCapacity);
    struct bucket_t * find(cache_key_t key, id receiver);

    static void bad_cache(id receiver, SEL sel, Class isa) __attribute__((noreturn));
};

struct bucket_t {
private:
    cache_key_t _key;
    IMP _imp;

public:
    inline cache_key_t key() const { return _key; }
    inline IMP imp() const { return (IMP)_imp; }
    inline void setKey(cache_key_t newKey) { _key = newKey; }
    inline void setImp(IMP newImp) { _imp = newImp; }

    void set(cache_key_t newKey, IMP newImp);
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

typedef struct category_t *Category;

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



1. 几个相关概念

   - AssociationsManager：其中包含一个静态变量 static AssociationsHashMap *map，保存所有关联对象的信息。是一个全局的 map。
   - AssociationsHashMap：全局 map ，保存所有关联对象的信息。
   - ObjectAssociationMap：保存一个 object 的所有关联对象信息。
   - ObjcAssociation：保存一条关联对象信息。
   - 以上概念的关系如下图，图片引用自 [文章](https://juejin.im/post/5af86b276fb9a07aa34a59e6)：

   ![1635a628a228e34](https://tva1.sinaimg.cn/large/006tNbRwly1gaetrcissjj30s80hwaed.jpg)

2. 源码实现：

   ```c
   // 获取关联对象的值。
   id _object_get_associative_reference(id object, void *key) {
       id value = nil;
       uintptr_t policy = OBJC_ASSOCIATION_ASSIGN;
       {
           AssociationsManager manager;
           AssociationsHashMap &associations(manager.associations());
           disguised_ptr_t disguised_object = DISGUISE(object);
           AssociationsHashMap::iterator i = associations.find(disguised_object);
           if (i != associations.end()) {
               ObjectAssociationMap *refs = i->second;
               ObjectAssociationMap::iterator j = refs->find(key);
               if (j != refs->end()) {
                   ObjcAssociation &entry = j->second;
                   value = entry.value();
                   policy = entry.policy();
                   if (policy & OBJC_ASSOCIATION_GETTER_RETAIN) ((id(*)(id, SEL))objc_msgSend)(value, SEL_retain);
               }
           }
       }
       if (value && (policy & OBJC_ASSOCIATION_GETTER_AUTORELEASE)) {
           ((id(*)(id, SEL))objc_msgSend)(value, SEL_autorelease);
       }
       return value;
   }
   
   // 设置关联对象的值。
   void _object_set_associative_reference(id object, void *key, id value, uintptr_t policy) {
       // retain the new value (if any) outside the lock.
       ObjcAssociation old_association(0, nil);
       id new_value = value ? acquireValue(value, policy) : nil;
       {
           AssociationsManager manager;
           AssociationsHashMap &associations(manager.associations());
           disguised_ptr_t disguised_object = DISGUISE(object);
           if (new_value) {
               // break any existing association.
               AssociationsHashMap::iterator i = associations.find(disguised_object);
               if (i != associations.end()) {
                   // secondary table exists
                   ObjectAssociationMap *refs = i->second;
                   ObjectAssociationMap::iterator j = refs->find(key);
                   if (j != refs->end()) {
                       old_association = j->second;
                       j->second = ObjcAssociation(policy, new_value);
                   } else {
                       (*refs)[key] = ObjcAssociation(policy, new_value);
                   }
               } else {
                   // create the new association (first time).
                   ObjectAssociationMap *refs = new ObjectAssociationMap;
                   associations[disguised_object] = refs;
                   (*refs)[key] = ObjcAssociation(policy, new_value);
                   object->setHasAssociatedObjects();
               }
           } else {
               // setting the association to nil breaks the association.
               AssociationsHashMap::iterator i = associations.find(disguised_object);
               if (i !=  associations.end()) {
                   ObjectAssociationMap *refs = i->second;
                   ObjectAssociationMap::iterator j = refs->find(key);
                   if (j != refs->end()) {
                       old_association = j->second;
                       refs->erase(j);
                   }
               }
           }
       }
       // release the old value (outside of the lock).
       if (old_association.hasValue()) ReleaseValue()(old_association);
   }
   ```

   - 上面的 map 都是使用了 c++ 的 unordered_map 来作为存储数据结构，所以没有处理 hash 碰撞的代码。

   - 通过之前的几个基本概念中图示的讲解，这里的代码就变得很清晰了，就是按照上图的数据结构，从外向内查找和插入。

   - 关联对象的存储是存在全局的 map 中，并没有存储在相应的 object 中。而在 object 销毁的时候，会清除掉全局 map 中相应的条目。参考代码如下：

     ```c
     /***********************************************************************
     * objc_destructInstance
     * Destroys an instance without freeing memory. 
     * Calls C++ destructors.
     * Calls ARR ivar cleanup.
     * Removes associative references.
     * Returns `obj`. Does nothing if `obj` is nil.
     * Be warned that GC DOES NOT CALL THIS. If you edit this, also edit finalize.
     * CoreFoundation and other clients do call this under GC.
     **********************************************************************/
     void *objc_destructInstance(id obj) 
     {
         if (obj) {
             // Read all of the flags at once for performance.
             bool cxx = obj->hasCxxDtor();
             bool assoc = !UseGC && obj->hasAssociatedObjects();
             bool dealloc = !UseGC;
     
             // This order is important.
             if (cxx) object_cxxDestruct(obj);
             if (assoc) _object_remove_assocations(obj);
             if (dealloc) obj->clearDeallocating();
         }
     
         return obj;
     }
     ```

   - 关联属性的内存标记有 assign/retaion/copy，但是没有 weak。

# 2. Runtime 整体流程概述

> 这部分参考了[这篇文章](https://juejin.im/post/5cfdbcb76fb9a07eb3096f01)。

```c
/***********************************************************************
* lookUpImpOrForward.
* The standard IMP lookup. 
* initialize==NO tries to avoid +initialize (but sometimes fails)
* cache==NO skips optimistic unlocked lookup (but uses cache elsewhere)
* Most callers should use initialize==YES and cache==YES.
* inst is an instance of cls or a subclass thereof, or nil if none is known. 
*   If cls is an un-initialized metaclass then a non-nil inst is faster.
* May return _objc_msgForward_impcache. IMPs destined for external use 
*   must be converted to _objc_msgForward or _objc_msgForward_stret.
*   If you don't want forwarding at all, use lookUpImpOrNil() instead.
**********************************************************************/
IMP lookUpImpOrForward(Class cls, SEL sel, id inst, 
                       bool initialize, bool cache, bool resolver)
{
    Class curClass;
    IMP imp = nil;
    Method meth;
    bool triedResolver = NO;

    runtimeLock.assertUnlocked();
    //============= 消息传递阶段开始 ==============

    // Optimistic cache lookup
    if (cache) {
        imp = cache_getImp(cls, sel);
        if (imp) return imp;
    }

    if (!cls->isRealized()) {
        rwlock_writer_t lock(runtimeLock);
        realizeClass(cls);
    }

    if (initialize  &&  !cls->isInitialized()) {
        _class_initialize (_class_getNonMetaClass(cls, inst));
        // If sel == initialize, _class_initialize will send +initialize and 
        // then the messenger will send +initialize again after this 
        // procedure finishes. Of course, if this is not being called 
        // from the messenger then it won't happen. 2778172
    }

    // The lock is held to make method-lookup + cache-fill atomic 
    // with respect to method addition. Otherwise, a category could 
    // be added but ignored indefinitely because the cache was re-filled 
    // with the old value after the cache flush on behalf of the category.
 retry:
    runtimeLock.read();

    // Ignore GC selectors
    if (ignoreSelector(sel)) {
        imp = _objc_ignored_method;
        cache_fill(cls, sel, imp, inst);
        goto done;
    }

    // Try this class's cache.

    imp = cache_getImp(cls, sel);
    if (imp) goto done;

    // Try this class's method lists.

    meth = getMethodNoSuper_nolock(cls, sel);
    if (meth) {
        log_and_fill_cache(cls, meth->imp, sel, inst, cls);
        imp = meth->imp;
        goto done;
    }

    // Try superclass caches and method lists.

    curClass = cls;
    while ((curClass = curClass->superclass)) {
        // Superclass cache.
        imp = cache_getImp(curClass, sel);
        if (imp) {
            if (imp != (IMP)_objc_msgForward_impcache) {
                // Found the method in a superclass. Cache it in this class.
                log_and_fill_cache(cls, imp, sel, inst, curClass);
                goto done;
            }
            else {
                // Found a forward:: entry in a superclass.
                // Stop searching, but don't cache yet; call method 
                // resolver for this class first.
                break;
            }
        }

        // Superclass method list.
        meth = getMethodNoSuper_nolock(curClass, sel);
        if (meth) {
            log_and_fill_cache(cls, meth->imp, sel, inst, curClass);
            imp = meth->imp;
            goto done;
        }
    }
    //============= 消息传递阶段结束 ==============
    
    //============= 消息解析阶段开始 ==============
    // No implementation found. Try method resolver once.

    if (resolver  &&  !triedResolver) {
        runtimeLock.unlockRead();
        _class_resolveMethod(cls, sel, inst);
        // Don't cache the result; we don't hold the lock so it may have 
        // changed already. Re-do the search from scratch instead.
        triedResolver = YES;
        goto retry;
    }
    //============= 消息解析阶段结束 ==============

    //============= 消息转发阶段开始 ==============
    // No implementation found, and method resolver didn't help. 
    // Use forwarding.

    imp = (IMP)_objc_msgForward_impcache;
    cache_fill(cls, sel, imp, inst);
    //============= 消息转发阶段结束 ==============

 done:
    runtimeLock.unlockRead();

    // paranoia: look for ignored selectors with non-ignored implementations
    assert(!(ignoreSelector(sel)  &&  imp != (IMP)&_objc_ignored_method));

    // paranoia: never let uncached leak out
    assert(imp != _objc_msgSend_uncached_impcache);

    return imp;
}
```



整体上消息的流程如下：

1. 进入消息传递阶段，判断消息接受者是否为 nil。
2. 利用 isa 指针找到自己的类对象。
3. 在类对象的 cache_t（方法缓存）中查找是否有方法，有则直接取出 bucket_t（桶）中 IMP（实现）。无则继续。
4. 在类对象的 method_list_t 中查找方法。有则直接取出，无则继续。
5. 找到类对象的 super_class ，继续在其父类中重复上面两个步骤进行查找。
6. 若一直往上都没有找到方法的实现，那么消息传递阶段结束，进入动态解析阶段。
7. 动态解析阶段，在这个阶段，若解析到方法，则结束。否则继续。
8. 最后会进入消息转发阶段，在这里可以指定别的类为自己实现这个方法。
9. 若上方步骤都没有找到方法的实现，则会报方法找不到的错误，无法识别消息，unrecognzied selector sent to instance。

整体大概分为 消息发送阶段、消息解析阶段、消息转发阶段，三个阶段。流程大概如下：

<img src="https://tva1.sinaimg.cn/large/006tNbRwly1gafnoq3bhkj31100nuwio.jpg" alt="image-20191231093856404" style="zoom:80%;" />





# 3. 消息传递

消息传递的流程如下图：

<img src="https://tva1.sinaimg.cn/large/006tNbRwly1gafp6kt3cpj30x60qwwjx.jpg" alt="image-20191231103042073" style="zoom:80%;" />



# 4. 消息解析

1. 消息解析的源码如下：

   ```c
   /***********************************************************************
   * _class_resolveMethod
   * Call +resolveClassMethod or +resolveInstanceMethod.
   * Returns nothing; any result would be potentially out-of-date already.
   * Does not check if the method already exists.
   **********************************************************************/
   void _class_resolveMethod(Class cls, SEL sel, id inst)
   {
       if (! cls->isMetaClass()) {
           // try [cls resolveInstanceMethod:sel]
           _class_resolveInstanceMethod(cls, sel, inst);
       } 
       else {
           // try [nonMetaClass resolveClassMethod:sel]
           // and [cls resolveInstanceMethod:sel]
           _class_resolveClassMethod(cls, sel, inst);
           if (!lookUpImpOrNil(cls, sel, inst, 
                               NO/*initialize*/, YES/*cache*/, NO/*resolver*/)) 
           {
               _class_resolveInstanceMethod(cls, sel, inst);
           }
       }
   }
   
   ```

   

2. 消息动态解析阶段的主要流程如下：

   <img src="https://tva1.sinaimg.cn/large/006tNbRwly1gafphbt2doj30ty0q4juq.jpg" alt="image-20191231104101394" style="zoom:80%;" />

2. 消息解析是通过两个 resolve 方法来实现动态解析的。参考代码如下：

   ```objective-c
   - (void)viewDidLoad {
       [super viewDidLoad];
       // Do any additional setup after loading the view, typically from a nib.
       //执行foo函数
       [self performSelector:@selector(foo:)];
   }
   
   + (BOOL)resolveInstanceMethod:(SEL)sel { //resolveClassMethod
       if (sel == @selector(foo:)) {//如果是执行foo函数，就动态解析，指定新的IMP
           class_addMethod([self class], sel, (IMP)fooMethod, "v@:"); // 通过 class_addMethod 方法，可以给某个 SEL 指定起实现 IMP。
           return YES;
       }
       return [super resolveInstanceMethod:sel];
   }
   
   void fooMethod(id obj, SEL _cmd) {
       NSLog(@"Doing foo");//新的foo函数
   }
   ```

# 5. 消息转发

1. 这部分主要是 `_objc_msgForward_impcache` 内实现，是汇编写的，就不贴源码了。
2. 消息转发主要有两部分组成：一是备用接收者，二是完整转发。
3. 备用接收者：
   1. 运行期系统会问reciever，能否把消息转给其他接收者来处理。
   2. reciever 通过 `- (id)forwardingTargetForSelector:(SEL)aSelector;` 方法来告诉是否有备用接收者，如果有则返回备用接收者，如果没有，则返回 nil。
4. 完整转发。
   1. runtime 向对象发送  `methodSignatureForSelector:` 消息，并取到返回的方法签名。
   2. 如果方法签名为空，则转发失败。如果不为空则，通过签名生成 NSInvocation 对象。
   3. 然后 runtime 通过 `forwardInvocation:` 方法，实现完整转发。
   4. 所以重写 `forwardInvocation:` 的同时也要重写 `methodSignatureForSelector:` 方法。
   5. 当一个对象由于没有相应的方法实现而无法响应某个消息时，运行时系统将通过 `forwardInvocation:` 消息通知该对象。每个对象都继承了 `forwardInvocation:` 方法，我们可以将消息转发给其它的对象。

# 6. Runtime 的应用

##### 1. Method Swizzling

> 

##### 2. 模拟多继承

>  通过备用接收者，即可以实现 多继承的效果。

# 7. Runtime 的相关问题

