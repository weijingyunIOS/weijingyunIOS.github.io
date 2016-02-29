---
layout: post
title: "SDWebImage源码解析---缓存管理"
date: 2016-01-27 16:57:41 +0800
comments: true
category:OpenSourceFrameWork
 
---

```
这篇文章主要是介绍了一下SDWebImage提供的一写关于缓存的其他功能，也提供了清除缓存的方法，这里粗略的介绍一下，希望对大家在以后的文件操作中有一定的帮助。

```

####磁盘空间的初始化方法

```
- (id)initWithNamespace:(NSString *)ns {
    // iOS使用的是沙盒机制，此处makeDiskCachePath就是获取Cache目录，并
    	在Cache目录下创建default目录
   // 比如我的mac上就显示/Users/poloby/Library/Developer/
   CoreSimulator/Devices/4404872F-4DDD-4AEA-AAD3-71BA1931D4C1/
   data/Containers/Data/Application/9C7E5D14-FBF0-41F1-A533-
   E8ACC59FCBAC/Library/Caches/default
   // 后面详解
    NSString *path = [self makeDiskCachePath:ns];
   // 最终的初始化，后面详解
return [self initWithNamespace:ns diskCacheDirectory:path];
}

```

makeDiskCachePath

```
-(NSString *)makeDiskCachePath:(NSString*)fullNamespace{
    // 获取当前用户应用下的Caches目录
    // 返回了一个包含用户Caches目录作为第一元素的数组，所以底下用的是
    	paths[0]
    // 即/Users/poloby/Library/Developer/CoreSimulator/Devices/
    4404872F-4DDD-4AEA-AAD3-71BA1931D4C1/data/Containers/Data/
    Application/9C7E5D14-FBF0-41F1-A533-E8ACC59FCBAC/Library/
    Caches/
    
    NSArray *paths = 
    NSSearchPathForDirectoriesInDomains(NSCachesDirectory, 
    								NSUserDomainMask, YES);
   // 在Caches目录下构建一个fullNamespace目录，此处默认是default目录
    return [paths[0] 
    		stringByAppendingPathComponent:fullNamespace];
}

```

initWithNamespace:diskCacheDirectory:

```
(id)initWithNamespace:(NSString *)ns diskCacheDirectory:
									(NSString *)directory {
									
    if ((self = [super init])) {
        // 再给Caches/default/后面加上fullNamspace
        // 最终可能获得的diskCachePath可能为
        NSString *fullNamespace = 
        [@"com.hackemist.SDWebImageCache."
        							 stringByAppendingString:ns];

        // 初始化kPNGSignatureData为PNG前8字节的标志：{0x89, 0x50,
        		 0x4E, 0x47, 0x0D, 0x0A, 0x1A, 0x0A}
        // 用于ImageDataHasPNGPreffix这个C函数中，判断该data是不是PNG
        	格式
        	
        kPNGSignatureData = [NSData 
        			dataWithBytes:kPNGSignatureBytes length:8];

        // 创建名为com.hackemist.SDWebImageCache的IO的串行队列
        
        _ioQueue =
         dispatch_queue_create("com.hackemist.SDWebImageCache", 
         							DISPATCH_QUEUE_SERIAL);

        // cache存储的最长时间为60 * 60 * 24 * 7，即一个星期
        _maxCacheAge = kDefaultCacheMaxCacheAge;

        // 注意此处不是直接使用[[NSCache alloc] init]进行初始化的，而
        	是使用了一个AutoPurgeCache
        // AutoPurgeCache和NSCache不同之处在于，如果AutoPurgeCache收
        	到一个内存警告，就会自动释放内存，调用NSCache的
        	removeAllObjects
        	
        _memCache = [[AutoPurgeCache alloc] init];
        _memCache.name = fullNamespace;

        // 初始化disk cache，一般情况下directory，除非你把Caches删除了
        if (directory != nil) {
            // 最终结果是/Users/poloby/Library/Developer/
            CoreSimulator/Devices/4404872F-4DDD-4AEA-
            AAD3-71BA1931D4C1/data/Containers/Data/Application/
            9C7E5D14-FBF0-41F1-A533-E8ACC59FCBAC/Library/Caches/
            default/com.hackemist.SDWebImageCache.default
            
            _diskCachePath = [directory 
            	stringByAppendingPathComponent:fullNamespace];
        } else {
            // 如果没有找到Caches目录，或者新建default目录失败。就重新
            	使用makeCachePath新建一个缓存目录
            NSString *path = [self makeDiskCachePath:ns];
            _diskCachePath = path;
        }

        // 默认需要解压缩图片
        _shouldDecompressImages = YES;
       // 新建一个NSFileManager也是放在ioQueue中的
        dispatch_sync(_ioQueue, ^{
            _fileManager = [NSFileManager new];
        });

	#if TARGET_OS_IPHONE
        // 订阅了app可能发生的时间
        // 出现内存警告
        (UIApplicationDidReceiveMemoryWarningNotification)，调用
        	clearMemory
        	
        [[NSNotificationCenter defaultCenter] addObserver:self
                           selector:@selector(clearMemory)                                                   		name:UIApplicationDidReceiveMemoryWarningNotification                           
                           object:nil];
                           
        // 程序终止(UIApplicationWillTerminateNotification)，调用
        	cleanDisk
        	
        [[NSNotificationCenter defaultCenter] addObserver:self
								selector:@selector(cleanDisk)
                name:UIApplicationWillTerminateNotification
                               object:nil];
        // 程序进入后台运行
        (UIApplicationDidEnterBackgroundNotification)，调用
        backgroundCleanDisk
        
        // backgroundCleanDisk就不赘述了，其实现了在后台注册了
        	cleanDiskWithCompletionBlock函数来处理后台的磁盘缓存
        	
        [[NSNotificationCenter defaultCenter] addObserver:self
						selector:@selector(backgroundCleanDisk)
				name:UIApplicationDidEnterBackgroundNotification
                       object:nil];
	#endif
    }

    return self;
}

```

####计算缓存文件的大小

```
- (void)calculateSizeWithCompletionBlock:(SDWebImageCalculateSizeBlock)completionBlock {
    NSURL *diskCacheURL = [NSURL fileURLWithPath:
    									self.diskCachePath
     									isDirectory:YES];

    dispatch_async(self.ioQueue, ^{
        NSUInteger fileCount = 0;
        NSUInteger totalSize = 0;
        
   NSDirectoryEnumerator *fileEnumerator = 
        			[_fileManager enumeratorAtURL:diskCacheURL
					includingPropertiesForKeys:@[NSFileSize]
					options: 
					 NSDirectoryEnumerationSkipsHiddenFiles
					 errorHandler:NULL];

   for (NSURL *fileURL in fileEnumerator) {
       NSNumber *fileSize;
       //获取单个文件大小的方法
       [fileURL getResourceValue:&fileSize 
       						forKey:NSURLFileSizeKey error:NULL];
       totalSize += [fileSize unsignedIntegerValue];
       //文件个数累加
       fileCount += 1;
   }

   if (completionBlock) {
       dispatch_async(dispatch_get_main_queue(), ^{
          completionBlock(fileCount, totalSize);
       });
    }
  });
}

```

####获取磁盘文件个数

```
- (NSUInteger)getDiskCount {
    __block NSUInteger count = 0;
    dispatch_sync(self.ioQueue, ^{
        NSDirectoryEnumerator *fileEnumerator = [_fileManager enumeratorAtPath:self.diskCachePath];
        count = [[fileEnumerator allObjects] count];
    });
    return count;
}

```

####SDWebImage定期清理缓存

```
- (void)cleanDiskWithCompletionBlock:
					(SDWebImageNoParamsBlock)completionBlock {
  dispatch_async(self.ioQueue, ^{
      // 这两个变量主要是为了下面生成NSDirectoryEnumerator准备的
      // 一个是记录遍历的文件目录，一个是记录遍历需要预先获取文件的哪些属性
      
        NSURL *diskCacheURL = [NSURL fileURLWithPath:
        									self.diskCachePath
        					 				isDirectory:YES];
        					 				
        NSArray *resourceKeys = @[NSURLIsDirectoryKey, 
        						NSURLContentModificationDateKey,
        						 NSURLTotalFileAllocatedSizeKey];

		
        // 递归地遍历diskCachePath这个文件夹中的所有目录，此处不是直接使
        	用diskCachePath，而是使用其生成的NSURL
        // 此处使用includingPropertiesForKeys:resourceKeys，这样每
        	个file的resourceKeys对应的属性也会在遍历时预先获取到
        NSDirectoryEnumerator *fileEnumerator =
        			 [_fileManager enumeratorAtURL:diskCacheURL
        			    includingPropertiesForKeys:resourceKeys
        		options:NSDirectoryEnumerationSkipsHiddenFiles
				errorHandler:NULL];


        // 获取文件的过期时间，SDWebImage中默认是一个星期
        // 不过这里虽然称*expirationDate为过期时间，但是实质上并不是这
        	样。
        // 其实是这样的，比如在2015/12/12/00:00:00最后一次修改文件，对
        	应的过期时间应该是
        // 2015/12/19/00:00:00，不过现在时间是2015/12/27/00:00:00，
        	我先将当前时间减去1个星期，得到
        // 2015/12/20/00:00:00，这个时间才是我们函数中的
        	expirationDate。
        // 用这个expirationDate和最后一次修改时间modificationDate比较
        	看谁更晚就行。
        	
        NSDate *expirationDate = [NSDate
        		dateWithTimeIntervalSinceNow:-self.maxCacheAge];
        
        NSMutableDictionary *cacheFiles = 
        						[NSMutableDictionary dictionary];
        						
        NSUInteger currentCacheSize = 0;

		// 在缓存的目录开始遍历文件.  此次遍历有两个目的:
        //
        //  1. 移除过期的文件
        //  2. 同时存储每个文件的属性（比如该file是否是文件夹、该file所
        		需磁盘大小，修改时间）
        
        NSMutableArray *urlsToDelete =
        						 [[NSMutableArray alloc] init];
        						 
        for (NSURL *fileURL in fileEnumerator) {
            NSDictionary *resourceValues = 
            	[fileURL resourceValuesForKeys:resourceKeys 
            							  error:NULL];

            // Skip directories.
            if ([resourceValues[NSURLIsDirectoryKey] boolValue]) 
            {
                continue;
            }

            // 移除过期文件
            // 这里判断过期的方式：对比文件的最后一次修改日期和
            	expirationDate谁更晚，如果expirationDate更晚，就认为
            	该文件已经过期，具体解释见上面
            NSDate *modificationDate = 
            	resourceValues[NSURLContentModificationDateKey];
            	
            if ([[modificationDate laterDate:expirationDate]
            				 isEqualToDate:expirationDate]) {
                [urlsToDelete addObject:fileURL];
                continue;
            }

          	// 计算当前已经使用的cache大小，
            // 并将对应file的属性存到cacheFiles中
            NSNumber *totalAllocatedSize =
            	 resourceValues[NSURLTotalFileAllocatedSizeKey];
            
            currentCacheSize += [totalAllocatedSize
             							unsignedIntegerValue];
            [cacheFiles setObject:resourceValues 
            									forKey:fileURL];
        }
        
        for (NSURL *fileURL in urlsToDelete) {
            [_fileManager removeItemAtURL:fileURL error:nil];
        }
	    // 如果我们当前cache的大小已经超过了允许配置的缓存大小，
	    	那就删除已经缓存的文件。
        // 删除策略就是，首先删除修改时间更早的缓存文件
         
        if (self.maxCacheSize > 0 
        	&& currentCacheSize > self.maxCacheSize)
        {
            // 直接将当前cache大小降到允许最大的cache大小的一般
            const NSUInteger desiredCacheSize = 
            							self.maxCacheSize / 2;

			// 根据文件修改时间来给所有缓存文件排序，按照修改时间越早越在
				前的规则排序            		
            NSArray *sortedFiles = [cacheFiles 
            	keysSortedByValueWithOptions:NSSortConcurrent
				usingComparator:^NSComparisonResult(id obj1, id 
				obj2) {
				 return [obj1[NSURLContentModificationDateKey]
				  		compare:
				 		obj2[NSURLContentModificationDateKey]];
             }];

            // 每次删除file后，就计算此时的cache的大小
            // 如果此时的cache大小已经降到期望的大小了，就停止删除文件了
            for (NSURL *fileURL in sortedFiles) {
                if ([_fileManager 
               		 removeItemAtURL:
                	fileURL error:nil])
                {
                    NSDictionary *resourceValues = 
                    						cacheFiles[fileURL];
                    // 根据resourceValues获取该文件所需磁盘空间大小
                    NSNumber *totalAllocatedSize = 
                resourceValues[NSURLTotalFileAllocatedSizeKey];
                	// 计算当前cache大小
                    currentCacheSize -= [totalAllocatedSize 
                    					  unsignedIntegerValue];

                    if (currentCacheSize < desiredCacheSize) {
                        break;
                    }
                }
            }
        }
        if (completionBlock) {
            dispatch_async(dispatch_get_main_queue(), ^{
                completionBlock();
            });
        }
    });
}


```
总结一下：

清理缓存的时机：

清除磁盘缓存
UIApplicationWillTerminateNotification
UIApplicationDidEnterBackgroundNotification

清除Cache缓存

UIApplicationDidReceiveMemoryWarningNotification
当收到这三个通知的时候回到用对应的清理缓存的方法

清理过程：

1、按照用户设置的maxCacheAge图片被缓存的最长时间

```
/**
 * The maximum length of time to keep an image in the cache, in seconds
 */
@property (assign, nonatomic) NSInteger maxCacheAge;
```
将所有modificationDate时间晚于maxCacheAge时间的文件删除

2、如果经过第一步的删除之后，图片的缓存仍然大于maxCacheSize,那么需要进一步的删除图片文件

```
/**
 * The maximum size of the cache, in bytes.
 */
@property (assign, nonatomic) NSUInteger maxCacheSize;

```
这一步的删除，最终的目的是将最终的缓存大小降低到maxCacheSize的一半，方式也是根据最后的修改时间先将所有的文件排序，然后依次删除，每一次删除都要确认是否当前的缓存大小是否小鱼maxCacheSize的一半

3、清理Cache缓存

```
- (void)clearMemory {
    [self.memCache removeAllObjects];
}

```

####手动清理磁盘图片缓存

直接清除，磁盘缓存目录下的所有文件

```
- (void)clearDiskOnCompletion:
					 (SDWebImageNoParamsBlock)completion
{
    dispatch_async(self.ioQueue, ^{
    	 // 先将存储在diskCachePath中缓存全部移除，然后新建一个空的
    	 	diskCachePath
        [_fileManager removeItemAtPath:self.diskCachePath 
        					      error:nil];
        					      
        [_fileManager createDirectoryAtPath:self.diskCachePath
                withIntermediateDirectories:YES
                                 attributes:nil
                                      error:NULL];

        if (completion) {
            dispatch_async(dispatch_get_main_queue(), ^{
                completion();
            });
        }
    });
}

```

####通过cacheKey获取某张图片

```
- (UIImage *)imageFromDiskCacheForKey:(NSString *)key {

    // First check the in-memory cache...
    UIImage *image = [self imageFromMemoryCacheForKey:key];
    if (image) {
        return image;
    }

    // Second check the disk cache...
    UIImage *diskImage = [self diskImageForKey:key];
    if (diskImage && self.shouldCacheImagesInMemory) {
        NSUInteger cost = SDCacheCostForImage(diskImage);
        [self.memCache setObject:diskImage forKey:key cost:cost];
    }

    return diskImage;
}

```

`注意：`这里在磁盘中找到这张图片之后，会将这张图片放到缓存中 用来表示他最近使用了

####通过搜索全路径获取图片数据(NSData)

```
- (NSData *)diskImageDataBySearchingAllPathsForKey:(NSString *)key {
    NSString *defaultPath = [self defaultCachePathForKey:key];
    NSData *data = [NSData dataWithContentsOfFile:defaultPath];
    if (data) {
        return data;
    }

    // fallback because of https://github.com/rs/SDWebImage/
    	pull/976 that added the extension to the disk file name
    // checking the key with and without the extension
    
    data = [NSData dataWithContentsOfFile:
    			[defaultPath stringByDeletingPathExtension]];
    			
    if (data) {
        return data;
    }

    NSArray *customPaths = [self.customPaths copy];
    for (NSString *path in customPaths) {
        NSString *filePath = [self cachePathForKey:key
        									 inPath:path];
        NSData *imageData = [NSData 
        						dataWithContentsOfFile:filePath];
        						
        if (imageData) {
            return imageData;
        }

        // fallback because of https://github.com/rs/SDWebImage/
        pull/976 that added the extension to the disk file name
        // checking the key with and without the extension
        
        imageData = [NSData dataWithContentsOfFile:
        			[filePath stringByDeletingPathExtension]];
        			
        if (imageData) {
            return imageData;
        }
    }

    return nil;
}

```
customPaths:是在搜索完缓存以及磁盘都没有找到的时候，查找一个只读的空间，判断图片是否存在


```
/**
 * Add a read-only cache path to search for images pre-cached by SDImageCache
 * Useful if you want to bundle pre-loaded images with your app
 *
 * @param path The path to use for this read-only cache path
 */
- (void)addReadOnlyCachePath:(NSString *)path;

```
当用户调用这个方法设置只读的路径时，会向customPaths中添加路径


####获取一张磁盘中缓存的图片（UIImage）

```
- (UIImage *)diskImageForKey:(NSString *)key {
    NSData *data = [self 
    			diskImageDataBySearchingAllPathsForKey:key];
    if (data) {
        UIImage *image = [UIImage sd_imageWithData:data];
        image = [self scaledImageForKey:key image:image];
        if (self.shouldDecompressImages) {
            image = [UIImage decodedImageWithImage:image];
        }
        return image;
    }
    else {
        return nil;
    }
}

```
从磁盘中取出图片要经历的几个过程：

1、取出NSData

2、将NSData转换为UIImage

3、做scale适配

4、图片是否需要解码，如果需要执行解码操作

5、返回这张图片



