---
layout: post
title: SDWebImage学习笔记
author: Arthur
date: 2016-07-27 14:51:01 +0800
tags: iOS vander
---

SDWebImage托管在github上[https://github.com/rs/SDWebImage](SDWebImage托管在github上。https://github.com/rs/SDWebImage)  

**SDWebImage**功能如下：  

* 提供UIImageView的一个分类，以支持网络图片的加载与缓存管理
* 一个异步的图片加载器
* 一个异步的内存+磁盘图片缓存
* 支持GIF图片
* 支持WebP图片
* 后台图片解压缩处理

主要优点

* 确保同一个URL的图片不被下载多次
* 确保虚假的URL不会被反复加载
* 确保下载及缓存时，主线程不被阻塞 


**主要代码解析：**  
下载的枚举类型   

```
typedef NS_OPTIONS(NSUInteger, SDWebImageDownloaderOptions) {
   
  SDWebImageDownloaderLowPriority = 1 << 0,
    
  SDWebImageDownloaderProgressiveDownload = 1 << 1,
    
  /**
   * By default, request prevent the of NSURLCache. With this flag, NSURLCache
   * is used with default policies.
   * 默认情况下请求，但是不实用NSURLCache，如果设置该选项，则以默认的缓存策略来使用NSURLCache
  */
  SDWebImageDownloaderUseNSURLCache = 1 << 2,
    
  /**
   * Call completion block with nil image/imageData if the image was read from NSURLCache
   * (to be combined with SDWebImageDownloaderUseNSURLCache ).
   * 如果从NSURLCache缓存中读取图片，则使用nil作为参数来调用完成block
   */
     
  SDWebImageDownloaderIgnoreCachedResponse = 1 << 3,
    
  /**
   * In iOS 4+, continue the download of the image if the app goes to background. This is achieved by asking the system for
   * extra time in background to let the request finish. If the background task expires the operation will be cancelled.
   * 在iOS4+中， 允许程序在后台下载图片， 该操作通过向系统申请额外的时间来完成后台下载，如果后台任务终止，则会被取消。
  */
     
  SDWebImageDownloaderContinueInBackground = 1 << 4,
    
  /**
   * Handles cookies stored in NSHTTPCookieStore by setting 
   * NSMutableURLRequest.HTTPShouldHandleCookies = YES;
   * 通过设置NSMutableURLRequest。httpshouldHandleCookies=YES来处理存储在NSHTTPCookieStore中的cookie
   */
     
  SDWebImageDownloaderHandleCookies = 1 << 5,
    
  /**
   * Enable to allow untrusted SSL ceriticates.
   * Useful for testing purposes. Use with caution in production.
   * 允许不受信任的ssl证书，主要用于测试目的
   */
     
  SDWebImageDownloaderAllowInvalidSSLCertificates = 1 << 6,
    
  /**
   * Put the image in the high priority queue.
   * 将图片下载防盗高优先级队列中
   */
  SDWebImageDownloaderHighPriority = 1 << 7,
}; 
```

下载顺序:

``` 
SDWebImage提供了两种下载顺序，一种是以队列方式（先进先出），一种是以栈的方式（后进先出）。 
typedef NS_ENUM (NSInteger, SDWebImageDownloaderExecutionOrder) { 
    /**
     * Default value. All download operations will execute in queue style (first-in-first-out).
     * 默认用队列的方式， 按照fifo的方式下载
     */
    SDWebImageDownloaderFIFOExecutionOrder,
    /**
     * All download operations will execute in stack style (last-in-first-out).
     * 以栈的方式
     */
    SDWebImageDownloaderLIFOExecutionOrder
};
```

下载任务管理器
SDWebImageDownloader下载管理器是一个单例类，主要负责图片的下载操作管理，图片的下载是放在一个NSOperationQueue操作队列中来完成的，它的声明如下：

```
@property (strong, nonatomic) NSOperationQueue *downloadQueue;  //使用NSOperationQueue操作队列来完成下来
默认情况下，队列最大的并发数是6，如果需要我们可以通过SDWebImageDownloader类的maxConcurrentDownloads属性来修改。
所有下载操作的网络响应序列化处理都放在一个自定义的并行调度队列中来处理，其声明及定义如下： 
@property (SDDispatchQueueSetterSementics, nonatomic) dispatch_queue_t barrierQueue;
- (id)init {
    if ((self = [super init])) {
        ...
        _barrierQueue = dispatch_queue_create("com.hackemist.SDWebImageDownloaderBarrierQueue", DISPATCH_QUEUE_CONCURRENT);
        ...
    }
    return self;
}
```

每一个图片的下载都会对应一些回调操作，如下载进度回调、下载结果回调等，这些回调操作是block形式来呈现，为此在SDWebImageDownloader.h中定义了几个block，

```
/**
 *  下载进度回调
 */
typedef void(^SDWebImageDownloaderProgressBlock)(NSInteger receivedSize, NSInteger expectedSize, NSData * data);
/**
 *  下载完成回调
 */
typedef void(^SDWebImageDownloaderCompletedBlock)(UIImage *image, NSData *data, NSError *error, BOOL finished);
/**
 *  Header过滤
 */
typedef NSDictionary *(^SDWebImageDownloaderHeadersFilterBlock)(NSURL *url, NSDictionary *headers);
```

图片下载的这些回调信息存储在SDWebImageDownloader类的URLCallbacks属性中，该属性是个字典，key为图片的URL地址，value是个数组，包含每个图片的多组回调信息，由于我们允许多个图片同事下载，因此可能会有多个线程对URLCallbacks同事操作，为了保证URLCallbacks操作的线程安全性，SDWebImageDownloader将这些操作作为一个任务放在barrierQueue队列中，并设置屏障来确保同一时间只有一个线程操作URLCallbacks属性，我们以添加操作为例， 

```
- (void)addProgressCallback:(SDWebImageDownloaderProgressBlock)progressBlock andCompletedBlock:(SDWebImageDownloaderCompletedBlock)completedBlock forURL:(NSURL *)url createCallback:(SDWebImageNoParamsBlock)createCallback {
 ...
 
    /**
     *  确保同一时间只有一个线程能对URLCallbacks进行操作,有兴趣的同学可以查阅下dispatch_barrier_sync方法相关的描述这里不一一细说
     */
    dispatch_barrier_sync(self.barrierQueue, ^{
 ...
        // Handle single download of simultaneous download request for the same URL
        // 处理同一url的同步下载请求的单个下载
        NSMutableArray *callbacksForURL = self.URLCallbacks[url];
        NSMutableDictionary *callbacks = [NSMutableDictionary new];
        if (progressBlock) callbacks[kProgressCallbackKey] = [progressBlock copy];
        if (completedBlock) callbacks[kCompletedCallbackKey] = [completedBlock copy];
        [callbacksForURL addObject:callbacks];
        self.URLCallbacks[url] = callbacksForURL;

        if (first) {
            createCallback();
        }
    });
}
```