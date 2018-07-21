---
layout:     post
title:      Lottie 和 SVGA 的性能指标对比
subtitle:   
date:       2018-07-21
author:     G
header-img: img/post-bg-debug.png
catalog: true
tags:
    - iOS
    - Android
    - 动画
    - lottie
    - svga
---

# 背景

最近项目中准备做直播，直播中不可或缺的一个重要部件就是礼物了。设计妹子们做了很多绚丽的礼物动画。那么我们要选择什么样的方式来播放这些动画呢？

经过一番调研，基本放弃了 Gif 合适和帧动画的方式。最终还在选择范围之内的，就剩下了 [Lottie](https://airbnb.design/lottie/) 和 [SVGA](http://svga.io/) 两种动画方式。

接下来我们将对同一动画，在安卓和 iOS 平台上，在以下四个方面进行对比：

- 文件大小
- 内存占用
- CPU 占用
- 流畅度

# 文件大小
下图是使用一个60帧的动画导出的 SVGA 和 Lottie 所对应的文件大小比较：

![](https://ws4.sinaimg.cn/large/006tNc79ly1fthl9526klj30dc0240sl.jpg)

# 内存 & CPU 占用

## 安卓

1. svga：
	![](https://ws1.sinaimg.cn/large/006tNc79gy1fthl6q1nx6j30wk08dt8m.jpg)
2. lottie：
	![](https://ws2.sinaimg.cn/large/006tNc79gy1fthl6bj7jqj30wo08rdfr.jpg)
	
> 以上数据是在现有项目中测试的，所以绝对值没有意义，只看对比即可。

## iOS

1. svga:
	
	![](https://ws2.sinaimg.cn/large/006tNc79ly1fthl9kh1lgj30es0akdg6.jpg)
	
2. lottie:
	
	![](https://ws1.sinaimg.cn/large/006tNc79gy1fthla7at8xj30ga0ao74s.jpg)

# 流畅度

## 安卓

1. lottie 在视觉上是有非常明显的卡顿，实际的掉帧情况如图：

	![](https://ws2.sinaimg.cn/large/006tNc79gy1fthl7wjo6lj30f00qowgg.jpg)

2. svga 在视觉上是流畅的，实际上的掉帧情况如图：

	![](https://ws1.sinaimg.cn/large/006tNc79gy1fthl8b4lwsj30u01hc157.jpg)

## iOS

- iOS 上无法看出掉帧的情况，实际上用 instruments 看，lottie 的实际帧率是50左右，svga 的帧率是60。

# 结语

![](https://ws2.sinaimg.cn/large/006tNc79ly1fthl8wmhk9j30ny0dymxf.jpg)