---
layout: post
title: GPUImage ‘Incomplete filter FBO: 36055’ bug修复记录
tags: 
 - bug 记录
 - 基础
---
## GPUImage ‘Incomplete filter FBO: 36055’ bug修复记录
最近在开发的过程中使用GPUImage导出视频时遇到一个bug，断言在`[GPUImageMovieWrite createDataFBO]` 方法中的`NSAssert(status == GL_FRAMEBUFFER_COMPLETE, @"Incomplete filter FBO: %d", status);`。

在google的过程中找到了[The current version, the video processing error at the beginning . · Issue #1276 · BradLarson/GPUImage · GitHub](https://github.com/BradLarson/GPUImage/issues/1276)。



## bug说明

这边遇到问题的视频是系统的录屏视频，并通过AVFoundation截取并生成视频前两秒的AVComposition/AVVideoComposition。

上面给出的解决方法是:

```objc
	AVAsset *anAsset = [AVAsset assetWithURL:videoURL];
	AVAssetTrack *assetTrack = [[anAsset tracksWithMediaType:AVMediaTypeVideo] objectAtIndex:0];
	_movieFile = [[GPUImageMovie alloc] initWithAsset:anAsset];
	_movieWriter = [[GPUImageMovieWriter alloc] initWithMovieURL:_URLInputVideoFile size:assetTrack.naturalSize];
	if ([[anAsset tracksWithMediaType:AVMediaTypeAudio] count] > 0){
	    _movieFile.audioEncodingTarget = _movieWriter;
	} else {//no audio
	    _movieFile.audioEncodingTarget = nil;
	}
```
但在我这里，这个方法就行不通了，因为项目里面的 `[anAsset tracksWithMediaType:AVMediaTypeAudio].count > 0`。所以我这里的情况就是，新生成的AVComposition是有AudioTrack的，但这AudioTrack偏偏就是没有音频数据的。

那既然判断AudioTracks.count不是真正的判定是否有音频数据，那么我们只要进一步读取AudioTrack中是否有音频数据那不就OK了？

VAssetTrack有一个segments的属性，返回track中所有的segment，而AVAssetTrackSegment有一个empty属性:

```objc
/* indicates whether the AVAssetTrackSegment is an empty segment */
@property (nonatomic, readonly, getter=isEmpty) BOOL empty;
```
但是在我这边，这个empty是为NO的，还是有数据的。

一路不通，可以走另外一条路，我们可以创建一个AVAssetReader，直接读取asset的audio数据，这个做法是OK的，但不怎么美观，太粗暴了，直接略过。

既然从源头过滤问题视频不行，那么我们就从GPUImageWriter中入手，先分析问题出现的原因，再去修复它。



## bug出现的原因

`Terminating app due to uncaught exception 'NSInternalInconsistencyException', reason: 'Incomplete filter FBO: 36055'`这是系统给出的bug原因，而36055代表的是*No images are attached to the framebuffer.*[Framebuffer.Status Enumeration](https://neslib.github.io/Ooogles.Net/html/0e1349ae-da69-6e5e-edd6-edd8523101f8.htm)，结合google回来的解决方案，是因为我们给GPUIMageMovie.audioEncodingTarget赋值而导致的问题，那么为什么设置一个audioEncodingTarget会导致创建FBO出现问题呢?

这个问题先放一边，将关注点放到GPUImageMovie导出的整个流程当中:

```objc
// GPUImageMovie.m
// 这里的方法都是将无关代码删减过后的代码，可以直接查看GPUImage源码

// 开始asset的数据获取
- (void)processAsset
{
    reader = [self createAssetReader]; // 创建一个reader
    /* ...设置videoOutput/audioOutput的一些属性... */
	 // 开始读取数据并导出 --- 将数据传给GPUImageMovieWriter
    __unsafe_unretained GPUImageMovie *weakSelf = self;
    if (synchronizedMovieWriter != nil) {
        [synchronizedMovieWriter setVideoInputReadyCallback:^{ // 设置writer的videoInput回调
            return [weakSelf readNextVideoFrameFromOutput:readerVideoTrackOutput];
        }];
        [synchronizedMovieWriter setAudioInputReadyCallback:^{ // 设置writer的audioInput回调
            return [weakSelf readNextAudioSampleFromOutput:readerAudioTrackOutput];
        }];
        [synchronizedMovieWriter enableSynchronizationCallbacks]; // writer开始写数据
    }
	 /* ...其他操作... */
} 
// 读取视频的下一帧
- (BOOL)readNextVideoFrameFromOutput:(AVAssetReaderOutput *)readerVideoTrackOutput;
{
	  // 判断当前reader的状态
    if (reader.status == AVAssetReaderStatusReading && ! videoEncodingIsFinished) {
			// 获取targetbuffer
        CMSampleBufferRef sampleBufferRef = [readerVideoTrackOutput copyNextSampleBuffer];
        if (sampleBufferRef)  {
			/*
				buffer的时间调整操作
				将targetbuffer传给下一个inputTarget
			*/
          return YES;
        } else {
            /* 判断状态并结束progress */
        }
    } else if (synchronizedMovieWriter != nil) {
        /* 判断状态并结束progress */
    }
    return NO;
}
// 读取音频
- (BOOL)readNextAudioSampleFromOutput:(AVAssetReaderOutput *)readerAudioTrackOutput;
{
		// 判断状态
    if (reader.status == AVAssetReaderStatusReading && ! audioEncodingIsFinished) {
			// 读取音频 
        CMSampleBufferRef audioSampleBufferRef = [readerAudioTrackOutput copyNextSampleBuffer];
        if (audioSampleBufferRef)
        {
            /* 
					属性的设置
					将targetBuffer传给audioEncodingTarget
			   */
            return YES;
        }
        else {
            /* 判断状态并结束progress */
        }
    }
    else if (synchronizedMovieWriter != nil) {
       /* 判断状态并结束progress */
    }
    return NO;
}
```
从上面看，这里的关键代码是writer的`enableSynchronizationCallbacks`，在这个方法中主导了callback的调用:
```objc
// GPUImageMovieWriter.m
// 这里的方法都是将无关代码删减过后的代码，可以直接查看GPUImage源码

- (void)enableSynchronizationCallbacks;
{
    if (videoInputReadyCallback != NULL) {
        if( assetWriter.status != AVAssetWriterStatusWriting ) {
            [assetWriter startWriting]; // 调用writer的startWriting方法
        }
        [assetWriterVideoInput requestMediaDataWhenReadyOnQueue:videoQueue usingBlock:^{
            /* ...一些操作... */
            while( assetWriterVideoInput.readyForMoreMediaData && ! _paused ) { // 循环并读取video数据
                if( videoInputReadyCallback && ! videoInputReadyCallback() && ! videoEncodingIsFinished ) {
                    if( assetWriter.status == AVAssetWriterStatusWriting && ! videoEncodingIsFinished ) {
								// video编码结束
                            videoEncodingIsFinished = YES;
                            [assetWriterVideoInput markAsFinished];
                        }
                }
            }
        }];
    }
    if (audioInputReadyCallback != NULL) {
        [assetWriterAudioInput requestMediaDataWhenReadyOnQueue:audioQueue usingBlock:^{
            /* ...一些操作... */
            while( assetWriterAudioInput.readyForMoreMediaData && ! _paused ) {
					// 循环并读取音频数据
                if( audioInputReadyCallback && ! audioInputReadyCallback() && ! audioEncodingIsFinished ) {
                    if( assetWriter.status == AVAssetWriterStatusWriting && ! audioEncodingIsFinished ) {
                            audioEncodingIsFinished = YES;
                            [assetWriterAudioInput markAsFinished];
                        }
                }
            }
        }];
    }        
}
```
看码说话，videoInput和audioInput的操作都是相差不大的，都是在block中加入target数据。

我们来看一下audioInput里面的代码，` audioInputReadyCallback && ! audioInputReadyCallback() && ! audioEncodingIsFinished`，如果我们的audio数据为空，那么就一定会进入这个if里面调用`[assetWriterAudioInput markAsFinished]`结束audio数据的输入，换句话来说就是GPUImageMovieWriter默认只要设置了audio相关的参数，就会默认audio一定有参数，并且当`audioInputReadyCallback`回调返回为NO就默认audio数据已输入完毕。

如果我们给video和audio的block打断点就可以知道audio的回调是比video的回调提前很多这是因为它们两个的数据处理量级都不一致，video需要处理的数据会更大。那么很明显地可以推断出，在第一帧video数据处理之前，`[assetWriterAudioInput markAsFinished]`就已经调用了。

我们回到断言出现的方法中:

``` objc
- (void)createDataFBO;
{
    /* ...一些处理... */
    if ([GPUImageContext supportsFastTextureUpload]) {
        CVPixelBufferPoolCreatePixelBuffer (NULL, [assetWriterPixelBufferInput pixelBufferPool], &renderTarget);
		  /* ...一些处理... */
    }
   /* ...一些处理... */
	  GLenum status = glCheckFramebufferStatus(GL_FRAMEBUFFER);
    NSAssert(status == GL_FRAMEBUFFER_COMPLETE, @"Incomplete filter FBO: %d", status);
}
```
`[assetWriterPixelBufferInput pixelBufferPool]`这个属性有一个特性就是当assetWriter.status != writing就会返回空，那么很明显地，这里因为*pixelBufferPool*为空而导致*renderTarget*创建失败，进而出现这个bug。

我们来总结一下这个bug的出现原因:

* 只要我们将*GPUImageMovie.audioEncodingTarget*赋值给*GPUImageMovieWriter*就会生成*AudioInput*，进而在开始写入数据时，因为audio的数据为空，所以在创建FBO之前调用了`[audioInput markAsFinished]`方法将*assetWriter*标志为结束输入，这个时候*assetWriter*的状态一直为失败，所以就导致了renderTarget创建失败，进而导致崩溃。

那么这个也说明为什么只要将*GPUImageMovie.audioEncodingTarget*赋值为nil就可以解决静音的bug。



## bug解决

在知道了bug出现的原因之后，解决的方法就有很多了，我们只要保证`createDataFBO`方法调用前`[audioInput markAsFinished]`不会调用就OK了。

在这里我选择继承*GPUImageMovieWriter*并重写`-enableSynchronizationCallbacks`方法，并创建了一个bool值来标志video是否已经开始progress:

``` objc
#import "GCImageMovieWriter.h"
#import <GPUImage/GPUImage.h>

@implementation GCImageMovieWriter {
    BOOL videoBeganEncoding, videoEncodingIsFinished, audioEncodingIsFinished;
    dispatch_queue_t audioQueue, videoQueue;
}

@synthesize videoInputReadyCallback;
@synthesize audioInputReadyCallback;
@synthesize paused = _paused;

- (void)enableSynchronizationCallbacks {
    ///!!!: 修复音频轨没有数据的bug
    // https://github.com/BradLarson/GPUImage/issues/1276
    videoBeganEncoding = NO;
    if (videoInputReadyCallback != NULL)
    {
        if( assetWriter.status != AVAssetWriterStatusWriting )
        {
            [assetWriter startWriting];
        }
        videoQueue = dispatch_queue_create("com.sunsetlakesoftware.GPUImage.videoReadingQueue", NULL);
        [assetWriterVideoInput requestMediaDataWhenReadyOnQueue:videoQueue usingBlock:^{
            if( _paused )
            {
                //NSLog(@"video requestMediaDataWhenReadyOnQueue paused");
                // if we don't sleep, we'll get called back almost immediately, chewing up CPU
                usleep(10000);
                return;
            }
            //NSLog(@"video requestMediaDataWhenReadyOnQueue begin");
            while( assetWriterVideoInput.readyForMoreMediaData && ! _paused )
            {
                BOOL hasVideoEncoding = videoInputReadyCallback();
                if (hasVideoEncoding) {
                    videoBeganEncoding = YES;
                }
                if( videoInputReadyCallback && ! hasVideoEncoding && ! videoEncodingIsFinished )
                {
                    runAsynchronouslyOnContextQueue(_movieWriterContext, ^{
                        if( assetWriter.status == AVAssetWriterStatusWriting && ! videoEncodingIsFinished )
                        {
                            videoEncodingIsFinished = YES;
                            [assetWriterVideoInput markAsFinished];
                        }
                    });
                }
            }
            //NSLog(@"video requestMediaDataWhenReadyOnQueue end");
        }];
    }
    
    if (audioInputReadyCallback != NULL)
    {
        audioQueue = dispatch_queue_create("com.sunsetlakesoftware.GPUImage.audioReadingQueue", NULL);
        [assetWriterAudioInput requestMediaDataWhenReadyOnQueue:audioQueue usingBlock:^{
            if( _paused )
            {
                //NSLog(@"audio requestMediaDataWhenReadyOnQueue paused");
                // if we don't sleep, we'll get called back almost immediately, chewing up CPU
                usleep(10000);
                return;
            }
            //NSLog(@"audio requestMediaDataWhenReadyOnQueue begin");
            while( assetWriterAudioInput.readyForMoreMediaData && ! _paused )
            {
                if( audioInputReadyCallback && ! audioInputReadyCallback() && ! audioEncodingIsFinished )
                {
                    runAsynchronouslyOnContextQueue(_movieWriterContext, ^{
                        if( assetWriter.status == AVAssetWriterStatusWriting && ! audioEncodingIsFinished && videoBeganEncoding )
                        {
                            audioEncodingIsFinished = YES;
                            [assetWriterAudioInput markAsFinished];
                        }
                    });
                }
            }
            //NSLog(@"audio requestMediaDataWhenReadyOnQueue end");
        }];
    }   
}
@end
```
