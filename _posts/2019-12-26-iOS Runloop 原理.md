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



# 1. Runloop 机制











# 参考文章

1. https://juejin.im/post/5aca2b0a6fb9a028d700e1f8
2. https://blog.ibireme.com/2015/05/18/runloop/



临时记录点：

1. CFRunLoopRef 是在 CoreFoundation 框架内的，它提供了纯 C 函数的 API，所有这些 API 都是线程安全的。NSRunLoop 是基于 CFRunLoopRef 的封装，提供了面向对象的 API，但是这些 API 不是线程安全的。
2. 