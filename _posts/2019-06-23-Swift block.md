---
layout:     post
title:      Swift Block
subtitle:   
date:       2019-06-23
author:     G
header-img: img/post-bg-swift.jpg
catalog: true
tags:
    - swift
    - block
	- closure
---



# 简介



Swift 中的闭包 Closure 和 OC 中的 block 基本是一个作用和地位。



# 定义

1. **闭包的表达式语法**
   闭包表达式语法有如下的一般形式：

   ```swift
     { (parameters/接收的参数) -> (return type/闭包返回值类型) in
           statements/保存在闭包中需要执行的代码
     }
   ```

   

2. **利用typealias为闭包类型定义别名**

   我们可以用 `typealias` 来为看似较为复杂的闭包类型定义别名，这样以后我们就可以用别名直接去申明这样类型的闭包了，例子如下：

   ```swift
   //为没有参数也没有返回值的闭包类型起一个别名
      typealias Nothing = () -> ()
       
    //如果闭包的没有返回值，那么我们还可以这样写，
      typealias Anything = () -> Void
       
    //为接受一个Int类型的参数不返回任何值的闭包类型 定义一个别名：PrintNumber
      typealias PrintNumber = (Int) -> ()
       
    //为接受两个Int类型的参数并且返回一个Int类型的值的闭包类型 定义一个别名：Add
      typealias Add = (Int, Int) -> (Int)
   ```

   

3. **简化写法**

   ```swift
   // 形式1: 带有参数参数类型，参数名，返回值类型
    sumClosure = { (a: Int, b: Int) -> Int in return a + b}
   
   // 形式2: 省略参数类型
   sumClosure = { (a,b) -> Int in return a + b}
   
   // 形式3: 省略返回值类型
   sumClosure = { (a,b) in return a + b}
   
   // 形式4：省略参数小括号
   sumClosure = { a,b in return a + b}
   
   // 形式5: 省略参数
   sumClosure = { return $0 + $1}
   
   // 形式6: 省略关键字
   returnsumClosure = { $0 + $1}
   
   ```

4. 



# 循环引用



# 逃逸





# Reference



1. https://www.jianshu.com/p/7c599b96815b