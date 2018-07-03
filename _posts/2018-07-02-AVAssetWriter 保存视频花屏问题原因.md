---
layout:     post
title:      AVAssetWriter 保存视频花屏问题原因
subtitle:   
date:       2018-07-02
author:     G
header-img: img/post-bg-2015.jpg
catalog: true
tags:
    - iOS
    - 视频
---

# 问题

在使用以下代码生成 MP4 视频的时候，最终生成出来的视频出现了花屏的问题。


```
NSError *error = nil;
AVAssetWriter *videoWriter = [[AVAssetWriter alloc] initWithURL:[NSURL fileURLWithPath:_localPath]
                                                       fileType:AVFileTypeMPEG4
                                                          error:&error];
NSParameterAssert(videoWriter);
    
NSDictionary *videoSettings = [NSDictionary dictionaryWithObjectsAndKeys:
                               AVVideoCodecH264, AVVideoCodecKey,
                               [NSNumber numberWithInt:350], AVVideoWidthKey,
                               [NSNumber numberWithInt:350], AVVideoHeightKey,
                               nil];
AVAssetWriterInput* writerInput = [AVAssetWriterInput assetWriterInputWithMediaType:AVMediaTypeVideo
                                                                     outputSettings:videoSettings];
    
NSParameterAssert(writerInput);
NSParameterAssert([videoWriter canAddInput:writerInput]);
[videoWriter addInput:writerInput];
    
    
    
AVAssetWriterInputPixelBufferAdaptor *adaptor =[AVAssetWriterInputPixelBufferAdaptor
                                                assetWriterInputPixelBufferAdaptorWithAssetWriterInput:writerInput
                                                sourcePixelBufferAttributes:nil];
[videoWriter startWriting];
[videoWriter startSessionAtSourceTime:kCMTimeZero];
```


# 原因和解决办法

- 原因：Video 的 Width 和 Height 在不是 16 的整数倍数的时候，就会出现花屏，具体原因有时间再调研。
- 解决办法：将 350 改为 16 倍数，比如 320 即可。
