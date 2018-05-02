---
layout: post
title: "NSURLSession 学习笔记"
date: 2015-11-14 14:25:59 +0800
comments: true
categories: iOS NSURLSession
---

###NSURLSession
> *NSURLSession是iOS7中新的网络接口。程序在前台时，NSURLSession与NSURLConnection可以互相替代工作。*

**功能**

1. 通过URL将数据下载到内存
2. 通过URL将数据下载到文件系统
3. 将数据上传到指定URL
4. 在`后台`完成上述功能

**可设置的工作模式**

* `默认会话模式（default）`：工作模式类似于原来的NSURLConnection，使用的是基于磁盘缓存的持久化策略，使用用户keychain中保存的证书进行认证授权。

* `瞬时会话模式（ephemeral）`：该模式不使用磁盘保存任何数据。所有和会话相关的caches，证书，cookies等都被保存在RAM中，因此当程序使会话无效，这些缓存的数据就会被自动清空。

* `后台会话模式（background）`：该模式在后台完成上传和下载，在创建Configuration对象的时候需要提供一个NSString类型的ID用于标识完成工作的后台会话。

**简单使用代码示范**

{% highlight objc %}
NSURLSessionConfiguration * config ;
config =[NSURLSessionConfiguration defaultSessionConfiguration];
    
NSURLSession * session ;
session =[NSURLSession sessionWithConfiguration:config
                                           delegate:self
                                      delegateQueue:[NSOperationQueue mainQueue]];
    
NSURLSessionDataTask * task ;
task =[session dataTaskWithURL:[NSURL URLWithString:@"http://www.baidu.com"]
             completionHandler:^(NSData * _Nullable data, NSURLResponse * _Nullable response, NSError * _Nullable error) {
                 NSString * dataStr =[[NSString alloc]initWithData:data encoding:NSUTF8StringEncoding];
                 NSLog(@"DATA:%@",dataStr);
    }];
[task resume];
{% endhighlight %}
<!--more-->
代码中除了NSURLSession，还用到 `NSURLSessionConfiguration ` `NSURLSessionDataTask ` ,下面分别介绍这两个类

**NSURLSessionConfiguration**

> *用于配置`NSURLSession`的属性*

**三种模式**

{% highlight objc %}
//默认会话模式（default）
+ (NSURLSessionConfiguration *)defaultSessionConfiguration;  
//瞬时会话模式（ephemeral）
+ (NSURLSessionConfiguration *)ephemeralSessionConfiguration;  
//后台会话模式（background）
///iOS9已弃用
+ (NSURLSessionConfiguration *)backgroundSessionConfiguration:(NSString *)identifier; 
///使用这个方法代替
+ (NSURLSessionConfiguration *)backgroundSessionConfigurationWithIdentifier:(NSString *)identifier;
{% endhighlight %}
三种对应前面介绍的`可设置的工作模式`

**属性**

{% highlight objc %}
//是否允许使用蜂窝链接
@property BOOL allowsCellularAccess;

//属性为YES时表示当程序在后台运作时由系统自己选择最佳的网络连接配置
//这个标志允许系统为分配任务进行性能优化。这意味着只有当设备有足够电量时，设备
//才通过Wifi进行数据传输。如果电量低，或者只仅有一个蜂窝连接，传输任务是不会运行的
//`后台传输最好选这个`
@property (getter=isDiscretionary) BOOL discretionary NS_AVAILABLE(NA, 7_0);  
{% endhighlight %}

**NSURLSessionTask**
> *基本网络任务类。有三个子类，分别用以获取数据、上传和下载。*

继承关系图

![](/pics/t1.png)

这三种类型的Task都是通过NSURLSession 的实例对象中的方法创建

**NSURLSessionDataTask**
>*可以上传数据，上传完成后再进行下载*

{% highlight objc %}
- (NSURLSessionDataTask *)dataTaskWithRequest:(NSURLRequest *)request;
- (NSURLSessionDataTask *)dataTaskWithURL:(NSURL *)url;

- (NSURLSessionDataTask *)dataTaskWithRequest:(NSURLRequest *)request completionHandler:(void (^)(NSData *data, NSURLResponse *response, NSError *error))completionHandler;  
- (NSURLSessionDataTask *)dataTaskWithURL:(NSURL *)url completionHandler:(void (^)(NSData *data, NSURLResponse *response, NSError *error))completionHandler;
{% endhighlight %}

**NSURLSessionUploadTask**
> *上传数据*

{% highlight objc %}
- (NSURLSessionUploadTask *)uploadTaskWithRequest:(NSURLRequest *)request fromFile:(NSURL *)fileURL;  
- (NSURLSessionUploadTask *)uploadTaskWithRequest:(NSURLRequest *)request fromData:(NSData *)bodyData;  
- (NSURLSessionUploadTask *)uploadTaskWithStreamedRequest:(NSURLRequest *)request; 

- (NSURLSessionUploadTask *)uploadTaskWithRequest:(NSURLRequest *)request fromFile:(NSURL *)fileURL completionHandler:(void (^)(NSData *data, NSURLResponse *response, NSError *error))completionHandler;   
- (NSURLSessionUploadTask *)uploadTaskWithRequest:(NSURLRequest *)request fromData:(NSData *)bodyData completionHandler:(void (^)(NSData *data, NSURLResponse *response, NSError *error))completionHandler;  
{% endhighlight %}

**NSURLSessionDownloadTask**
> *下载数据，支持断点续传*

{% highlight objc %}
- (NSURLSessionDownloadTask *)downloadTaskWithRequest:(NSURLRequest *)request;  
- (NSURLSessionDownloadTask *)downloadTaskWithURL:(NSURL *)url;
//断点续传
- (NSURLSessionDownloadTask *)downloadTaskWithResumeData:(NSData *)resumeData; 

- (NSURLSessionDownloadTask *)downloadTaskWithRequest:(NSURLRequest *)request completionHandler:(void (^)(NSURL *location, NSURLResponse *response, NSError *error))completionHandler; 
- (NSURLSessionDownloadTask *)downloadTaskWithURL:(NSURL *)url completionHandler:(void (^)(NSURL *location, NSURLResponse *response, NSError *error))completionHandler;
- (NSURLSessionDownloadTask *)downloadTaskWithResumeData:(NSData *)resumeData completionHandler:(void (^)(NSURL *location, NSURLResponse *response, NSError *error))completionHandler;  
{% endhighlight %}


**TASK Delegate**

* NSURLSessionTaskDelegate
 
{% highlight objc %}
//当收到302重定向时调用
//调用completionHandler() block,传 `newRequest` 变量时重定向
//传递`nil`时取消重定向。
//默认是重定向的
- (void)URLSession:(NSURLSession *)session task:(NSURLSessionTask *)task
                     willPerformHTTPRedirection:(NSHTTPURLResponse *)response
                                     newRequest:(NSURLRequest *)request
                              completionHandler:(void (^)(NSURLRequest * __nullable))completionHandler;
                              
//当收到身份验证质询时调用                              
- (void)URLSession:(NSURLSession *)session task:(NSURLSessionTask *)task
                            didReceiveChallenge:(NSURLAuthenticationChallenge *)challenge 
                              completionHandler:(void (^)(NSURLSessionAuthChallengeDisposition disposition, NSURLCredential * __nullable credential))completionHandler;
                              
//当需要一个新的body stream时，
//This may be necessary when authentication has failed for any request that involves a body stream. 
- (void)URLSession:(NSURLSession *)session task:(NSURLSessionTask *)task
                              needNewBodyStream:(void (^)(NSInputStream * __nullable bodyStream))completionHandler;
                              
//定期调用以通知delegate 上传进度
- (void)URLSession:(NSURLSession *)session task:(NSURLSessionTask *)task
                                didSendBodyData:(int64_t)bytesSent
                                 totalBytesSent:(int64_t)totalBytesSent
                       totalBytesExpectedToSend:(int64_t)totalBytesExpectedToSend;
                       
//当任务完成时调用
//error可能为nil，如果为nil,则表明任务完成
- (void)URLSession:(NSURLSession *)session task:(NSURLSessionTask *)task
                           didCompleteWithError:(nullable NSError *)error;
                           
{% endhighlight %}

* NSURLSessionDataDelegate
 
{% highlight objc %}
//当收到 response 时调用，
//并且在completionHandler 调用之前 不会继续收取更多的数据
//你可以 用 disposition对象 取消这个请求
//或者将 data task 转换到 download task
- (void)URLSession:(NSURLSession *)session dataTask:(NSURLSessionDataTask *)dataTask
                                 didReceiveResponse:(NSURLResponse *)response
                                  completionHandler:(void (^)(NSURLSessionResponseDisposition disposition))completionHandler;
                                  

//data task 将转成 download task 
//并且不会再上传更多的数据                                  
- (void)URLSession:(NSURLSession *)session dataTask:(NSURLSessionDataTask *)dataTask
                              didBecomeDownloadTask:(NSURLSessionDownloadTask *)downloadTask;
- (void)URLSession:(NSURLSession *)session dataTask:(NSURLSessionDataTask *)dataTask
                                didBecomeStreamTask:(NSURLSessionStreamTask *)streamTask;
                                
//收完数据时调用
//数据可能是不连续的                                
- (void)URLSession:(NSURLSession *)session dataTask:(NSURLSessionDataTask *)dataTask
                                     didReceiveData:(NSData *)data;
                                     
//response 将缓存时调用
//给 completionHandler 传nil时，不缓存                                     
- (void)URLSession:(NSURLSession *)session dataTask:(NSURLSessionDataTask *)dataTask
                                  willCacheResponse:(NSCachedURLResponse *)proposedResponse 
                                  completionHandler:(void (^)(NSCachedURLResponse * __nullable cachedResponse))completionHandler;

{% endhighlight %}

* NSURLSessionDownloadDelegate

{% highlight objc %}

- (void)URLSession:(NSURLSession *)session downloadTask:(NSURLSessionDownloadTask *)downloadTask
                              didFinishDownloadingToURL:(NSURL *)location;
- (void)URLSession:(NSURLSession *)session downloadTask:(NSURLSessionDownloadTask *)downloadTask
                                           didWriteData:(int64_t)bytesWritten
                                      totalBytesWritten:(int64_t)totalBytesWritten
                              totalBytesExpectedToWrite:(int64_t)totalBytesExpectedToWrite;
- (void)URLSession:(NSURLSession *)session downloadTask:(NSURLSessionDownloadTask *)downloadTask
                                      didResumeAtOffset:(int64_t)fileOffset
                                     expectedTotalBytes:(int64_t)expectedTotalBytes;

{% endhighlight %}
