---
layout: post
title: 通过PHPhotoLibrary获取UIImage内存不释放的bug
tags:
 - bug记录
 - 基础
---

## bug简述
现在有这么一个场景：
* 需要从相册中获取数量不定的图片，然后需要将获取到的图片转换成视频，因为现在照片有可能需要在iCloud上下载，所以不能获取一张图片之后直接生成视频，而是需要将所有选中的图片获取之后并记录在数组当中，当所有图片获取完成再逐张图片生成视频，在所有操作都完成之后就将图片释放。
* 获取图片使用的方法是:

```objc
- (PHImageRequestID)requestImageForAsset:(PHAsset *)asset 
                              targetSize:(CGSize)targetSize 
                             contentMode:(PHImageContentMode)contentMode 
                                 options:(nullable PHImageRequestOptions *)options 
                           resultHandler:(void (^)(UIImage *__nullable result, NSDictionary *__nullable info))resultHandler;
```

* 流程： 选中图片(PHAsset) --> 循环选中图片数组(PHAsset)获取图片 --> 将获取到的图片都保存到一个数组中 --> 循环image数组生成视频 --> 释放image数组

bug：
* 调用PHPhotoLibrary的方法获取图片时，图片将解码并将数据存在内存当中，这时根据我们的逻辑将对UIImage进行强引用，并在完成图片转视频的操作之后将UIImage给释放掉。
* 在我的设想中，这时这部分的内存就应该自动回收，但我发现在iPhone内存为2g以上的机器这部分的内存都不会自动回收(iPhone 6是会回收的)，并且当App进入到后台的时候这部分内存也会自动回收(这就表明我们的这部分代码是没有内存泄漏的)。
* 这应该是UIImage的缓存机制而引起的(太坑了)。

## bug解决方案
既然知道是因为UIImage缓存机制引起的bug，那么只要避免引起UIImage的缓存机制就OK了。如果我们不对UIImage强引用，Image是会正常释放掉内存的，所以这边在获取图片的方法改成了:
```objc
- (PHImageRequestID)requestImageDataForAsset:(PHAsset *)asset 
                                     options:(nullable PHImageRequestOptions *)options 
                               resultHandler:(void(^)(NSData *__nullable imageData, NSString *__nullable dataUTI,UIImageOrientation orientation, NSDictionary *__nullable info))resultHandler;
```
并在数组中对imageData强引用，这时内存是平稳的，并没有突增很大的内存(UIImage没有解码)，然后在图片生成视频前才将image创建出来，并且没有对image进行强引用(在创建UIImage的时候内存也没有增加，应该是没有对图片进行解码)。在视频生成之后，中间解码图片所产生的内存都随着类的释放而释放。
到目前为止，这个内存bug应该算是解决了，但为什么使用`[UIImage imageWithData:]`就没有进行缓存，这跟我看到的一篇博文解释不一样 [iOS 处理图片的一些小 Tip](https://blog.ibireme.com/2015/11/02/ios_image_tips/)
* 通过数据创建 UIImage 时，UIImage 底层是调用 ImageIO 的 CGImageSourceCreateWithData() 方法。该方法有个参数叫 ShouldCache，在 64 位的设备上，这个参数是默认开启的。这个图片也是同样在第一次显示到屏幕时才会被解码，随后解码数据被缓存到 CGImage 内部。与 imageNamed 创建的图片不同，如果这个图片被释放掉，其内部的解码数据也会被立刻释放。
