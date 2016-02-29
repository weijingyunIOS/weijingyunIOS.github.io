---
layout: post
title: SDWebImage源码解析-----下载
date: 2016-01-27 14:40:28 +0800
comments: true
category:OpenSourceFrameWork
 
---

##图片下载

###外层方法调用
####最外层方法的调用
首先，看一下使用SD进行图片下载时，调用的代码：

```
[cell.imageView sd_setImageWithURL:
			[NSURL URLWithString:
						[_objects objectAtIndex:indexPath.row]]
            placeholderImage:[UIImage imageNamed:@"placeholder"] 
            options:indexPath.row == 0 ?
				SDWebImageRefreshCached : 0];

```
很简单，需要的参数只有URL、placeholderImage和options三个,从字面的意思就可以看到
URL:要下载图片的URL
placeholderImage：要显示的占位图
options:下载时候的一些条件设置

####非必要参数的包装

方法：

```
- (void)sd_setImageWithURL:(NSURL *)url placeholderImage:(UIImage *)placeholder options:(SDWebImageOptions)options progress:(SDWebImageDownloaderProgressBlock)progressBlock completed:(SDWebImageCompletionBlock)completedBlock

```
在最外面调用的基础上增加了两个参数
progress:下载进度跟进的回调
completed:下载完成的进度的回调

下面从这个方法开始，对于图片的下载过程做一个详细的分析


###开始下载操作（包括缓存的查找）

####取消现有的图片的下载：

```
 [self sd_cancelCurrentImageLoad];

```
看一下具体实现


```									
- (void)sd_cancelImageLoadOperationWithKey:(NSString *)key {
    // 取消下载队列中正在进行的操作 这个方法后面会介绍
    NSMutableDictionary *operationDictionary = 
    								[self operationDictionary];
    //防止重复下载							
    id operations = [operationDictionary objectForKey:key];
    
    if (operations) {
        if ([operations isKindOfClass:[NSArray class]]) {
            for (id <SDWebImageOperation> operation
                                               in operations)
             {
                if (operation) {
                    [operation cancel];
                }
            }
        } else if ([operations  conformsToProtocol:
        						@protocol(SDWebImageOperation)])
        {
            [(id<SDWebImageOperation>) operations cancel];
        }
        //将这个任务移除
        [operationDictionary removeObjectForKey:key];
    }
}

```
这个方法是在每一次图片下载开始之前，如果存在正在下载的任务，那么现将这个任务取消，目的是为了防止重复下载。

获取当前的操作

```
- (NSMutableDictionary *)operationDictionary {
    NSMutableDictionary *operations = 
    		objc_getAssociatedObject(self, &loadOperationKey);
    		
    if (operations) {
        return operations;
    }
    
    operations = [NSMutableDictionary dictionary];
    
    objc_setAssociatedObject(self, &loadOperationKey,
    		 operations, OBJC_ASSOCIATION_RETAIN_NONATOMIC);
    		 
    return operations;
}

```


`问题`：这里为啥非要operations是一个遵守的SDWebImageOperation的对象？

<font color = red>这里希望大家指导一下</font>

`问题`:operations对象什么情况下会是一个数组？

<font color = red>这里希望大家指导一下</font>

`问题：`

每次下载之前，都把原来的下载给暂停了，是不是以为这一次只能下载一张图片？

解答：并不是这样的，这个方法首先是写在了UIView的分类中的，每一个UIView以及他的子类（UIButton或者UIImageView）都拥有一个operationDictionary，每次之前取消，可以理解为一个UIImageview保证对应一个URL

与取消对应的添加操作：


```
- (void)sd_setImageLoadOperation:(id)operation forKey:(NSString *)key {
    [self sd_cancelImageLoadOperationWithKey:key];
    NSMutableDictionary *operationDictionary = [self 
    									operationDictionary];
    [operationDictionary setObject:operation forKey:key];
}
```


####占位图片的设置

```
    if (!(options & SDWebImageDelayPlaceholder)) {
        dispatch_main_async_safe(^{
            self.image = placeholder;
        });
    }
    
    /**
     * By default, placeholder images are loaded while the image is loading. This flag will delay the loading
     * of the placeholder image until after the image has finished loading.
     */
    //默认情况，占位图片在图片加载过程中会显示
    //如果设置了这个属性，将会延迟占位图片的加载时间（图片加载完成之后）
    
    SDWebImageDelayPlaceholder = 1 << 9,
    主要是设置占位图片显示的时机

```

如果没有设置为SDWebImageDelayPlaceholder那么立即设置图片的占位图，那么如果设置了SDWebImageDelayPlaceholder这个属性之后什么时间设置占位图片呢

```
if ((options & SDWebImageDelayPlaceholder)) {
     wself.image = placeholder;
     [wself setNeedsLayout];
}

```
在图片下载完成之后，如果下载失败image = nil 时，会将占位图片设置


占位图片设置完成后，就开始了图片的加载，也就是本片文章的重头戏，图片缓存中查找或者图片的下载

####创建一个新的下载操作

```
 id <SDWebImageOperation> operation = 			
 		[SDWebImageManager.sharedManager downloadImageWithURL:url 
 		options:options 
 		progress:progressBlock 
 		completed:^(UIImage *image, NSError *error, 
 					SDImageCacheType cacheType, 
 					BOOL finished, NSURL *imageURL)

```
创建一个新的下载操作，并将下载操作添加到下载队列中

```
  //添加到下载队列中
  [self sd_setImageLoadOperation:operation 
  							forKey:@"UIImageViewImageLoad"];

```

下面的代码是在

```
id <SDWebImageOperation> operation =
	 	[SDWebImageManager.sharedManager 		
	 					downloadImageWithURL:url 
	 					options:options
	 					progress:progressBlock 
	 					completed:^(UIImage *image, NSError
	 					 *error, SDImageCacheType cacheType, BOOL
	 					 finished, NSURL *imageURL)
{
	//图片下载完成之后的操作
});

```

#### 操作的执行

```
/**
 * 如果缓存中没有给定URL对应的图片，下载这张图片，否则返回缓存的图片
 *
 * @param url            要下载图片的URL
 * @param options        图片下载的request的选项设置
 * @param progressBlock  下载进度监听的回调
 * @param completedBlock 下载完成的回调（必要的参数不可为nil）
 *
 *   completedBlock
 *   
 */
	当使用SDWebImageProgressiveDownload时，参数finished被设置为NO，图片下载的过程中这个回调将会被调用很多次返回的是一个部分图片，当图片完整下载完成的时候这个回调会在最后调用一下，设置一个完整的图片，同事将参数finished设置为YES

- (id <SDWebImageOperation>)downloadImageWithURL:(NSURL *)url
                                         options:(SDWebImageOptions)options
                                        progress:(SDWebImageDownloaderProgressBlock)progressBlock
                                       completed:(SDWebImageCompletionWithFinishedBlock)completedBlock;

```

两个补充的参数：

```
completedBlock的类型

typedef void(^SDWebImageCompletionWithFinishedBlock)
 				(UIImage *image, NSError *error, 
 				SDImageCacheType cacheType,
 	 			BOOL finished, NSURL *imageURL);

SDImageCacheType类型：

	typedef NS_ENUM(NSInteger, SDImageCacheType) {
    /**
     * The image wasn't available the SDWebImage caches, but was
     	 downloaded from the web.
     */
    SDImageCacheTypeNone,
    /**
     * The image was obtained from the disk cache.
     */
    SDImageCacheTypeDisk,
    /**
     * The image was obtained from the memory cache.
     */
    SDImageCacheTypeMemory
}

```

####必要的参数completedBlock

```
    // Invoking this method without a completedBlock is 
    	pointless
    NSAssert(completedBlock != nil,
    		 @"If you mean to prefetch the image, use -
    		 [SDWebImagePrefetcher prefetchURLs] instead");

```
如果传入的completedBlock是nil,将会报错！！！！！
解释：如果没有completedBlock，将没办法给imageview设置图片，所以下面的代码就没有意义了

[断言学习入门](http://www.cnblogs.com/moondark/archive/2012/03/12/2392315.html)


####对URL做特殊处理

由于在某些时候，出于某些特殊的原因，Xcode不会对这里的类型匹配做出警告的提示，所以作者在这里增加了一个容错处理，允许这里传递一个字符串类型的URL

```
    if ([url isKindOfClass:NSString.class]) {
        url = [NSURL URLWithString:(NSString *)url];
    }

    // Prevents app crashing on argument type error like sending NSNull instead of NSURL
    if (![url isKindOfClass:NSURL.class]) {
        url = nil;
    }

```

####再次新建一个操作
 新建了一个SDWebImageCombinedOperation对象

```
__block SDWebImageCombinedOperation *operation = 
    				[SDWebImageCombinedOperation new];
    				
__weak SDWebImageCombinedOperation *weakOperation = operation;

```

这里看一下SDWebImageCombinedOperation:这是一个遵守了SDWebImageOperation协议的对象继承自NSObject，包含了一个属性cacheOperation为NSOperation类型

```
@interface SDWebImageCombinedOperation : NSObject 
										<SDWebImageOperation>

@property (assign, nonatomic, getter = isCancelled) BOOL 
												     cancelled;
												     
@property (copy, nonatomic) SDWebImageNoParamsBlock cancelBlock;

@property (strong, nonatomic) NSOperation *cacheOperation;

@end

```

`注意`：downloadImageWithURL返回的这个id类型（遵守SDWebImageOperation协议）的对象


####黑名单处理

判断当前要下载的图片的URL是否存在于黑名单中

```
  BOOL isFailedUrl = NO;
    @synchronized (self.failedURLs) {
        isFailedUrl = [self.failedURLs containsObject:url];
    }

```

####URL是否合法

```
    if (url.absoluteString.length == 0 
    				|| (!(options & SDWebImageRetryFailed)
     				&& isFailedUrl)) 
    {
        dispatch_main_sync_safe(^{
            NSError *error = [NSError 
            	errorWithDomain:NSURLErrorDomain 
            	code:NSURLErrorFileDoesNotExist 
            	userInfo:nil];
            completedBlock(nil, error, SDImageCacheTypeNone, YES, url);
        });
        return operation;
    }

```

[如何自定义错误](http://blog.sina.com.cn/s/blog_71715bf801019ymq.html)


####将操作添加到正在进行的队列中

```
    @synchronized (self.runningOperations) {
        [self.runningOperations addObject:operation];
    }

```

`@synchronized`:

```
　　@synchronized，代表这个方法加锁, 相当于不管哪一个线程（例如线程A），运行到这个方法时,都要检查有没有其它线程例如B正在用这个方法，有的话要等正在使用synchronized方法的线程B运行完这个方法后再运行此线程A,没有的话,直接运行。它包括两种用法：synchronized 方法和 synchronized 块。

@synchronized 方法控制对类（一般在IOS中用在单例中）的访问：每个类实例对应一把锁，每个 synchronized 方法都必须获得调用该方法锁方能执行，否则所属就会发生线程阻塞，方法一旦执行，就独占该锁，直到从该方法返回时才将锁释放，此后被阻塞的线程方能获得该锁，重新进入可执行状态。这种机制确保了同一时刻对于每一个类，至多只有一个处于可执行状态，从而有效避免了类成员变量的访问冲突（只要所有可能访问类的方法均被声明为 synchronized）

```

####获取cachekey

NSString *key = [self cacheKeyForURL:url];

```
- (NSString *)cacheKeyForURL:(NSURL *)url {
    if (self.cacheKeyFilter) {
        return self.cacheKeyFilter(url);
    }
    else {
        return [url absoluteString];
    }
}

```

`self.cacheKeyFilter`是什么东西?

```
//cache filter 在SDWebImageManager每次将URL转换为cache key的时候调用，功能是去掉图片URL中的动态部分
[[SDWebImageManager sharedManager] setCacheKeyFilter:^(NSURL *url) {
    url = [[NSURL alloc] initWithScheme:url.scheme host:url.host path:url.path];
    return [url absoluteString];
}];

 * @endcode
 */
 
 @property (nonatomic, copy) SDWebImageCacheKeyFilterBlock cacheKeyFilter;

```
cache filter 是一个SDWebImageManager每次需要讲一个URL转换为cachekey时会调用的一个方法，用来去掉图片URL中的动态部分

####图片缓存

查找这张图片有没有被缓存过，具体的缓存查找策略会在后面单独写一篇文章

```
- (NSOperation *)queryDiskCacheForKey:(NSString *)key done:(SDWebImageQueryCompletedBlock)doneBlock

```

####图片下载

`下面就开始了正式的“下载” 根据第一部分的介绍，我们知道，在每次下载之前，我们都会先去缓存中查找一下，看是否存在，所以这里可以先跳到缓存看一下`

如果，这张图片没有被缓存过，那么就要开始下载这张图片

####图片要从网络获取的条件

```
if (operation.isCancelled) {
    @synchronized (self.runningOperations) {
    	[self.runningOperations removeObject:operation];
    }

    return;
}

```
这里的operation是一个SDWebImageCombinedOperation，具体内容跟上面的那个差不多，就不赘述了

####图片下载的条件

```
if ((!image || options & SDWebImageRefreshCached) 
&& (![self.delegate respondsToSelector:
			@selector(imageManager:shouldDownloadImageForURL:)] || [self.delegate imageManager:self 
			shouldDownloadImageForURL:url]))

```
下面来好好分析一下这个条件：
!image 图片存在 即缓存中没有找到
options & SDWebImageRefreshCached:图片找到了是否需要跟新缓存（重新下载）

是否实现了imageManager:shouldDownloadImageForURL:方法
[self.delegate respondsToSelector:
			@selector(imageManager:shouldDownloadImageForURL:)
如果实现了 就执行这个方法
[self.delegate imageManager:self shouldDownloadImageForURL:url]

#####小插曲
imageManager:shouldDownloadImageForURL:是干啥的？

```
/**
 * Controls which image should be downloaded when the image is not found in the cache.
 *
 * @param imageManager The current `SDWebImageManager`
 * @param imageURL     The url of the image to be downloaded
 *
 * @return Return NO to prevent the downloading of the image on cache misses. If not implemented, YES is implied.
 */
 //当缓存中图片不存在的时候，用这个方法来判断是否需要下载这张图片
 //返回NO来阻止图片的下载，如果没实现默认返回yes（始终下载）
- (BOOL)imageManager:(SDWebImageManager *)imageManager shouldDownloadImageForURL:(NSURL *)imageURL;

```

这里 我们队这整个的判断做一下分析：
如果图片没有缓存或者需要更新缓存 || 如果没实现代理方法则下载，实现代理方法则根据代理方法的返回值进行判断


判断好条件，接下来我们继续看

####是否需要更新缓存

```
if (image && options & SDWebImageRefreshCached) {
   dispatch_main_sync_safe(^{
		//缓存中已经有这张图片了，但是因为要更新缓存，所以会有重新下载
		//这样是NSURLCache有机会从服务器端刷新自身缓存。
       completedBlock(image, nil, cacheType, YES, url);
   });
}

```
如果缓存中有这张图片，但是用户设置了需要更新缓存，那么因为已经有这种图片了，所以直接调用completedBlock去显示图片，同时也会继续往后走去下载图片来更新缓存

####图片下载的参数设置

```
if (options & SDWebImageProgressiveDownload) 
downloaderOptions |= SDWebImageDownloaderProgressiveDownload;

if (options & SDWebImageRefreshCached)
 downloaderOptions |= SDWebImageDownloaderUseNSURLCache;
 
if (options & SDWebImageContinueInBackground)
 downloaderOptions |= SDWebImageDownloaderContinueInBackground;
 
if (options & SDWebImageHandleCookies)
 downloaderOptions |= SDWebImageDownloaderHandleCookies;
 
if (options & SDWebImageAllowInvalidSSLCertificates) 
	downloaderOptions |= 		
			SDWebImageDownloaderAllowInvalidSSLCertificates;
			
if (options & SDWebImageHighPriority) 
		downloaderOptions |= SDWebImageDownloaderHighPriority;
		
if (image && options & SDWebImageRefreshCached) {
     //如果设置了SDWebImageRefreshCached就不能设置SDWebImageDownloaderProgressiveDownload属性，因为，SDWebImageRefreshCached说明缓存中存在了，图片会理解执行completion方法
     downloaderOptions &= 
     					~SDWebImageDownloaderProgressiveDownload;

     //如果图片下载设置了SDWebImageRefreshCached就必须设置SDWebImageDownloaderIgnoreCachedResponse 为了保证更新NSURLCache
     downloaderOptions |= 
     			SDWebImageDownloaderIgnoreCachedResponse;
}
            

```

####开始下载

```
id <SDWebImageOperation> subOperation = [self.imageDownloader downloadImageWithURL:url options:downloaderOptions progress:progressBlock completed:^(UIImage *downloadedImage, NSData *data, NSError *error, BOOL finished)

```
这里又新起了一个遵守SDWebImageOperation协议的subOperation


下面，我们到这个方法里面去看一下：

```
 __block SDWebImageDownloaderOperation *operation;
 __weak __typeof(self)wself = self;

```
这里创建的operation是我们图片下载的主力SDWebImageDownloaderOperation


####添加进度监听

```
  - (void)addProgressCallback:(SDWebImageDownloaderProgressBlock)progressBlock completedBlock:(SDWebImageDownloaderCompletedBlock)completedBlock forURL:(NSURL *)url createCallback:(SDWebImageNoParamsBlock)createCallback {
    //URL不能为nil，因为在URLCallbacks中URL要作为key值，所以URL不可以为nil,如果为nil那么立即调用completedBlock返回土片数据为nil
    if (url == nil) {
        if (completedBlock != nil) {
            completedBlock(nil, nil, nil, NO);
        }
        return;
    }
	
	//一个图片的下载progressBlock可能会被执行很多次，而且很多次的执行可能并不是在同一个线程中，所以这里使用一个数组来保存
    dispatch_barrier_sync(self.barrierQueue, ^{
        BOOL first = NO;
        //数组保存
        if (!self.URLCallbacks[url]) {
            self.URLCallbacks[url] = [NSMutableArray new];
            first = YES;
        }

        NSMutableArray *callbacksForURL = 
        							self.URLCallbacks[url];
        NSMutableDictionary *callbacks =
        						 [NSMutableDictionary new];
        						 
        if (progressBlock)
         callbacks[kProgressCallbackKey] = [progressBlock copy];
         
        if (completedBlock)
       callbacks[kCompletedCallbackKey] = [completedBlock copy];
       
        [callbacksForURL addObject:callbacks];
        self.URLCallbacks[url] = callbacksForURL;
		//每张图片只有在第一次回调的时候会执行createCallback（）；
        if (first) {
            createCallback();
        }
    });
}
```

可以通过下面这张图片来了解结构层次：
![层次结构](http://images2015.cnblogs.com/blog/715314/201512/715314-20151204161948408-1934962430.png)

`问题`：

同一个URL,为什么在self.URLCallbacks中对应的是一个数组？有没有可能会造成控件的浪费？
解答：暂时还不太清楚 先往后面看看


####createCallback回调内容

```
NSTimeInterval timeoutInterval = wself.downloadTimeout;
if (timeoutInterval == 0.0) {
     timeoutInterval = 15.0;
}

```
如果用户没有主动设置下载超时timeoutInterval，则默认为15

#####创建图片下载的请求

```
NSMutableURLRequest *request = [[NSMutableURLRequest alloc] 
					initWithURL:url
 					cachePolicy:
 				  (options & SDWebImageDownloaderUseNSURLCache ?
					NSURLRequestUseProtocolCachePolicy : 
					NSURLRequestReloadIgnoringLocalCacheData) 
					timeoutInterval:timeoutInterval];
 				  
```
这里主要是设置了缓存策略和超时的时长，其中缓存策略：
options & SDWebImageDownloaderUseNSURLCache ?
					NSURLRequestUseProtocolCachePolicy : 
					NSURLRequestReloadIgnoringLocalCacheData
					
是否设置了SDWebImageDownloaderUseNSURLCache属性，并根据这个属性的设置设置NSURLRequest的缓存策略

```
  	//默认情况下不使用NSURLCache，可以通过设置这个属性使用NSURLCache
    SDWebImageDownloaderUseNSURLCache = 1 << 2,

```

三种策略关系的简述：

```
	SDWebImageDownloaderUseNSURLCache：在SDWebImage中，缺省情况下，request是不使用NSURLCache的，但是若使用该选项，就默认使用NSURLCache默认的缓存策略：NSURLRequestUseProtocolCachePolicy。

	NSURLRequestUseProtocolCachePolicy：对特定的 URL 请求使用网络协议（如HTTP）中实现的缓存逻辑。这是默认的策略。该策略表示如果缓存不存在，直接从服务端获取。
	如果缓存存在，会根据response中的Cache-Control字段判断 下一步操作，如: Cache-Control字段为must-revalidata, 则 询问服务端该数据是否有更新，无更新话 直接返回给用户缓存数据，若已更新，则请求服务端.
	NSURLRequestReloadIgnoringLocalCacheData：数据需要从原始地址(一般就是重新从服务器获取)加载。不使用现有缓存

```



#####请求的其他参数设置

```
 request.HTTPShouldHandleCookies = (options &
 						 SDWebImageDownloaderHandleCookies);

```

```
	// 如果设置HTTPShouldHandleCookies为YES，就处理存储在
		NSHTTPCookieStore中的cookies。
	// HTTPShouldHandleCookies表示是否应该给request设置cookie并随
		request一起发送出去。
    SDWebImageDownloaderHandleCookies = 1 << 5,

```

```
// HTTPShouldUsePipelining表示receiver(理解为iOS客户端)的下一个信息是否必须等到上一个请求回复才能发送。
// 如果为YES表示可以，NO表示必须等receiver收到先前的回复才能发送下个信息。

request.HTTPShouldUsePipelining = YES;
```

#####Header的设置

```
if (wself.headersFilter) {
    request.allHTTPHeaderFields = wself.headersFilter(url,
    								 [wself.HTTPHeaders copy]);
  } else {
            request.allHTTPHeaderFields = wself.HTTPHeaders;
        }

```

```
 //设置一个过滤器来设置每一个下载图片的HTTP请求的请求头
 //这个代码块在每一个图片的下载请求中都会被调用，返回一个可以用作HTTP请求header的字典
@property (nonatomic, copy) 
		SDWebImageDownloaderHeadersFilterBlock headersFilter;

```

```
// 简单看下HTTPHeader的初始化部分（如果下载webp图片，需要的header不一样）：
// #ifdef SD_WEBP
// _HTTPHeaders = [@{@"Accept": @"image/webp,image/*;q=0.8"} 
	mutableCopy];
// #else
// _HTTPHeaders = [@{@"Accept": @"image/*;q=0.8"} mutableCopy];
// #endif

```

#####图片下载完成后是否需要解码

```
shouldDecompressImages:图片下载完成是否需要解码

```
`注意`：解压缩已经下载的图片或者在缓存中的图片，可以提高性能，但是会耗费很多空间，缺省情况下是要解压缩图片。


#####身份验证

```

urlCredential:

	web 服务可以在返回 http 响应时附带认证要求的challenge，作用是询问 http 请求的发起方是谁，这时发起方应提供正确的用户名和密码（即认证信息），然后 web 服务才会返回真正的 http 响应。

	收到认证要求时，NSURLConnection 的委托对象会收到相应的消息并得到一个 NSURLAuthenticationChallenge 实例。该实例的发送方遵守 
	
	NSURLAuthenticationChallengeSender 协议。为了继续收到真实的数据，需要向该发送方向发回一个 NSURLCredential 实例

if (wself.urlCredential) {
     operation.credential = wself.urlCredential;
} else if (wself.username && wself.password) {
     operation.credential = [NSURLCredential 
     						credentialWithUser:wself.username
      								   password:wself.password 			persistence:NSURLCredentialPersistenceForSession];
}

```
NSURLCredentialPersistenceForSession表示在应用终止时，丢弃相应的 credential 。

[urlCredential详情参考这里](http://blog.csdn.net/majiakun1/article/details/17013379)


#####下载的优先级

```
if (options & SDWebImageDownloaderHighPriority) {
    operation.queuePriority = NSOperationQueuePriorityHigh;
  } else if (options & SDWebImageDownloaderLowPriority) {
    operation.queuePriority = NSOperationQueuePriorityLow;
  }

```
这个就不做过多的解释了，一看就能明白

#####添加到下载队列

```
  [wself.downloadQueue addOperation:operation];

```
这里的downloadQueue是一个NSOperationQueue

#####设置操作队列的执行顺序

```
if (wself.executionOrder == 
					SDWebImageDownloaderLIFOExecutionOrder) {
    // Emulate LIFO execution order by systematically adding new 
    	operations as last operation's dependency
    [wself.lastAddedOperation addDependency:operation];
     wself.lastAddedOperation = operation;
}

```
如果是先进先出，队列的结构，后来新加入的队列自然后在后面执行，如果是后进先出，栈结构 需要设置操作的依赖


说到这里，下面就剩下正式的发送请求

####下载请求

```
这里的参数应该是都比较清楚了，不在赘述
- (id)initWithRequest:(NSURLRequest *)request
              options:(SDWebImageDownloaderOptions)options
             progress:(SDWebImageDownloaderProgressBlock)progressBlock
            completed:(SDWebImageDownloaderCompletedBlock)completedBlock
            cancelled:(SDWebImageNoParamsBlock)cancelBlock

```
这个方法的返回值也是也operation用户可以定义，默认是SDWebImageDownloaderOperation


####图片下载progressBlock

```
progress:^(NSInteger receivedSize, NSInteger expectedSize) {
                                                             	SDWebImageDownloader *sself = wself;
	if (!sself) return;
                                                             	__block NSArray *callbacksForURL;
                                                             	dispatch_sync(sself.barrierQueue, ^{
                                                                 		callbacksForURL = [sself.URLCallbacks[url] copy];
                                                            
    });
    
   for (NSDictionary *callbacks in callbacksForURL) {
                                                                 		dispatch_async(dispatch_get_main_queue(), ^{
                                                                     			SDWebImageDownloaderProgressBlock callback = 
                                                                     							callbacks[kProgressCallbackKey];
                                                         	if (callback) callback(receivedSize, expectedSize);
                                                                 });

```
`注意：`
这里每一个URL对应的callbacksForURL是一个数组，数组中存放的是针对同一个URL的下载过程中的一些操作，这里对于同一个URL数组中的所有内容都做更新。

下载完成的回调(执行回调，删除回调的保存)

```
                                                        completed:^(UIImage *image, NSData *data, NSError *error, BOOL finished) {
                                                            SDWebImageDownloader *sself = wself;
if (!sself) return;                                                     __block NSArray *callbacksForURL;                                                dispatch_barrier_sync(sself.barrierQueue, ^{
	callbacksForURL = [sself.URLCallbacks[url] copy];
	if (finished) {
		[sself.URLCallbacks removeObjectForKey:url];
	}
});
    for (NSDictionary *callbacks in callbacksForURL) {
		SDWebImageDownloaderCompletedBlock callback 
						=callbacks[kCompletedCallbackKey];
                                                                if (callback) callback(image, data, error, finished);
 }
 }

```

下载操作被取消的回调：

```
                                                                                                                cancelled:^{
                                                            SDWebImageDownloader *sself = wself;
if (!sself) return;                                             dispatch_barrier_async(sself.barrierQueue, ^{
	[sself.URLCallbacks removeObjectForKey:url];
});
}];

```

到目前为止，除了具体的下载细节，图片下载就基本完成了！！！开始执行下载完成的回调

#####图片下载完成（包含成功或者失败）：

这里要注意下，突然发现当前存在着好几个operation，先通过下面这张图片，理一下层次结构

![icon](http://images2015.cnblogs.com/blog/715314/201512/715314-20151226140749593-1190538283.png)

现在回到了SDWebImageDownloader的downloadImageWithURL:


```
 __strong __typeof(weakOperation) strongOperation =
 												 weakOperation;
 												 
 if (!strongOperation || strongOperation.isCancelled) {
  // Do nothing if the operation was cancelled
  // See #699 for more details
  // if we would call the completedBlock, there could be a race
   condition between this block and another completedBlock for
    the same object, so if this one is called second, we will 
    overwrite the new data
 }

```

`问题`：

这里的解释是：如果操作被取消了，就不执行任何操作。如果我们调用了completedBlock，那么在同一个对象的另一个completedBlock之间就存在一个竞争关系，因此，如果这个方法被重复调用两次，我们会重写新的数据

####图片下载失败处理

确定下载失败的原因，触发下载完成的回调（返回必要信息），同时加入黑名单

```
dispatch_main_sync_safe(^{
  if (strongOperation && !strongOperation.isCancelled) {
      completedBlock(nil, error, SDImageCacheTypeNone,
       finished, url);
     }
 });

 if (error.code != NSURLErrorNotConnectedToInternet
     && error.code != NSURLErrorCancelled
     && error.code != NSURLErrorTimedOut
     && error.code != NSURLErrorInternationalRoamingOff
     && error.code != NSURLErrorDataNotAllowed
     && error.code != NSURLErrorCannotFindHost
     && error.code != NSURLErrorCannotConnectToHost) {
     @synchronized (self.failedURLs) {
          [self.failedURLs addObject:url];
     }
}

```

`注意：这里不管用户是否设置，都会将这个下载失败的URL加入到黑名单中`

####图片下载成功
判断是否为下载失败后的重试,如果是重试，那么就将这个图片从黑名单中移除

```
if ((options & SDWebImageRetryFailed)) {
   @synchronized (self.failedURLs) {
       [self.failedURLs removeObject:url];
    }
}

```

是否设置了SDWebImageRefreshCached（相同的URL 图片改变之后是否更新缓存）

```
if (options & SDWebImageRefreshCached && 
							image && !downloadedImage) {
   // Image refresh hit the NSURLCache cache, do not call the
    completion block
}

```

图片下载完成之后，是否需要对图片进行相应的处理（根据SDWebImageTransformAnimatedImage属性，用户可进行设置）：

先看一下图片下载成功之后判断条件

```
(downloadedImage && (!downloadedImage.images || (options & SDWebImageTransformAnimatedImage)) && [self.delegate respondsToSelector:@selector(imageManager:transformDownloadedImage:withURL:)]) 

```
downloadedImage:图片下载成功

图片不是gif且需要在图片下载完成之后，对图片进行其他操作
(!downloadedImage.images || (options & 		
				SDWebImageTransformAnimatedImage))


是否实现了图片下载完成之后形变的代理方法

[self.delegate respondsToSelector:@selector(imageManager:transformDownloadedImage:withURL:)

如果满足了上面的三个条件，表明在图片下载完成之后需要对图片进行形变：

```
如果要进行形变，这个代理方法需要开发者实现
UIImage *transformedImage = [self.delegate imageManager:self 
		transformDownloadedImage:downloadedImage withURL:url];

```

```
  //完成形变操作，且图片下载完成（图片没下载完成回到用这个方法吗）
 if (transformedImage && finished) {
     BOOL imageWasTransformed =
     			 ![transformedImage isEqual:downloadedImage];
     //图片保存
     [self.imageCache storeImage:transformedImage 
     		 recalculateFromImage:imageWasTransformed
      				imageData:(imageWasTransformed ? nil : data)
      				forKey:key toDisk:cacheOnDisk];
}

```

调用下载完成的回调

```
dispatch_main_sync_safe(^{
	//operation 没有被取消
    if (strongOperation && !strongOperation.isCancelled) {
		completedBlock(transformedImage, nil, 	
						SDImageCacheTypeNone, finished, url);
    }
});

```

`如果不需要再图片下载完成之后，对图片进行其他的处理,那么就做一下图片的缓存同时调用图片下载完成的回调即可`


#####图片下载完成最后移除

```
if (finished) {
   @synchronized (self.runningOperations) {
   if (strongOperation) {
     [self.runningOperations removeObject:strongOperation];
   }
 }
}

```


####操作的取消

```
operation.cancelBlock = ^{
	//首先取消SDWebImageDownloaderOperation
   [subOperation cancel];
                
   @synchronized (self.runningOperations) {
    __strong __typeof(weakOperation) strongOperation = 
    											weakOperation;
    if (strongOperation) {
    	//移除这一操作
        [self.runningOperations removeObject:strongOperation];
    }
 }
};

```
`注意`：这里取消的是operation而不是subOperation

operation:是一个NSOperation类型的
subOperation：是一个SDWebImageDownloaderOperation类型的

下面我们来仔细的看一下取消操作：

 [subOperation cancel];

这是SDWebImageDownloaderOperation种重写的取消

```
- (void)cancel {
    @synchronized (self) {
        if (self.thread) {
            [self 
            performSelector:@selector(cancelInternalAndStop) 
            onThread:self.thread withObject:nil 
            					  waitUntilDone:NO];
        }
        else {
            [self cancelInternal];
        }
    }
}

```
仔细观察一下这里的取消操作，会根据是否存在self.thread执行两个方法：

cancelInternalAndStop和cancelInternal

比较一下这两个方法可以看到cancelInternalAndStop只是比多了一句：

```
    CFRunLoopStop(CFRunLoopGetCurrent());
    
```
用来停止当前线程的运行循环。


####没找到图片：

```
dispatch_main_sync_safe(^{
   __strong __typeof(weakOperation) strongOperation = 
   												weakOperation;
   if (strongOperation && !weakOperation.isCancelled) {
       completedBlock(nil, nil, SDImageCacheTypeNone, YES, url);
   }
});
@synchronized (self.runningOperations) {
    [self.runningOperations removeObject:operation];
}

```

####图片在查找到或者下载完成之后图片的显示

SDWebImageAvoidAutoSetImage 图片下载完成之后是否立即设置给ImageView
在这里如果想要对图片的显示添加一些动画，我们可以从这里着手

```
	//下载完成之后是否自动设置图片
    SDWebImageAvoidAutoSetImage = 1 << 11

```

图片显示：

```
dispatch_main_sync_safe(^{
  if (!wself) return;
  //用户是否设置了需要手动设置图片的显示
  if (image 
  		&& (options & SDWebImageAvoidAutoSetImage)
   		&& completedBlock)
  {
      completedBlock(image, error, cacheType, url);
      return;
  }
   else if (image) {
      wself.image = image;
      [wself setNeedsLayout];
   } else {
   	  //图片下载失败延时显示占位图片
      if ((options & SDWebImageDelayPlaceholder)) {
            wself.image = placeholder;
            [wself setNeedsLayout];
      }
  }
  //图片下载成功之后的回调
  if (completedBlock && finished) {
       completedBlock(image, error, cacheType, url);
    }
  });

```
