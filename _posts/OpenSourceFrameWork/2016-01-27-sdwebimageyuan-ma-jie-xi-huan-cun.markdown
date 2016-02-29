---
layout: post
title: "SDWebImage源码解析------缓存"
date: 2016-01-27 14:44:12 +0800
comments: true
category: OpenSourceFrameWork
 
---
### SDWebImage缓存机制

#### 简介

SDWebImage的缓存可以说是这个框架的一个重大的有点，下面我们就来了解一下这个框架的缓存是如何实现的。

从整理来说，SDWebImage的缓存分为两部分SDImageCache使用NSCache实现，另一部分磁盘缓存，使用NSFileManager实现


#### 下载后的图片缓存

图片下载成功之后的缓存

```
- (void)storeImage:(UIImage *)image recalculateFromImage:(BOOL)recalculate imageData:(NSData *)imageData forKey:(NSString *)key toDisk:(BOOL)toDisk

```
#### 1、图片下载成功之后缓存到Cache中

```
@property (strong, nonatomic) NSCache *memCache;

    // if memory cache is enabled
    if (self.shouldCacheImagesInMemory) {
        NSUInteger cost = SDCacheCostForImage(image);
        [self.memCache setObject:image forKey:key cost:cost];
    }

```
self.memCache实际上就是一个NSCache,类似于字典的一种存储方式，需要传入图片的消耗

#### 2、图片下载成功之后缓存到磁盘

`注意：`这里在图片保存之前有一个图片格式转换的过程（*->PNG）

```
// 如果image存在，但是需要重新计算(recalculate)或者data为空
// 那就要根据image重新生成新的data
// 不过要是连image也为空的话，那就别存了
if (image && (recalculate || !data)) {
//#if TARGET_OS_IPHONE
    // 我们需要判断image是PNG还是JPEG
    // PNG的图片很容易检测出来，因为它们有一个特定的标示 (http://
    		www.w3.org/TR/PNG-Structure.html)
    // PNG图片的前8个字节不许符合下面这些值(十进制表示)
    // 137 80 78 71 13 10 26 10

    // 如果imageData为空l (举个例子，比如image在下载后需要transform，
    		那么就imageData就会为空)
    // 并且image有一个alpha通道, 我们将该image看做PNG以避免透明度
    		(alpha)的丢失（因为JPEG没有透明色）
	//通过AlphaInfo获取图片的先关信息
	int alphaInfo = CGImageGetAlphaInfo(image.CGImage);
	// 该image中确实有透明信息，就认为image为PNG
    BOOL hasAlpha = !(alphaInfo == kCGImageAlphaNone ||
			  alphaInfo == kCGImageAlphaNoneSkipFirst ||
              alphaInfo == kCGImageAlphaNoneSkipLast);
    //疑问：是否为PNG格式的图片和透明度有什么关系
    BOOL imageIsPng = hasAlpha;

	// 但是如果我们已经有了imageData，我们就可以直接根据data中前几个字节
		判断是不是PNG
   if ([imageData length] >= [kPNGSignatureData length]) {
   		// ImageDataHasPNGPreffix就是为了判断imageData前8个字节
   			是不是符合PNG标志
      imageIsPng = ImageDataHasPNGPreffix(imageData);
   }
   //格式转换
   if (imageIsPng) {
       data = UIImagePNGRepresentation(image);
   }
    else {
        data = UIImageJPEGRepresentation(image, (CGFloat)1.0);
   }
//#else
    data = [NSBitmapImageRep representationOfImageRepsInArray:
    				image.representations 
    				usingType: NSJPEGFileType properties:nil];
//#endif
}


```

图片个格式处理转换完成之后，将这个图片保存到本地

```

// 首先判断disk cache的文件路径是否存在，不存在的话就创建一个
// disk cache的文件路径是存储在_diskCachePath中的
if (![_fileManager fileExistsAtPath:_diskCachePath]) {
    [_fileManager createDirectoryAtPath:_diskCachePath 
    		withIntermediateDirectories:YES 
    				attributes:nil error:NULL];
}

 // get cache Path for image key
 获取图片保存的路径：
 NSString *fullNamespace = [@"com.hackemist.SDWebImageCache." stringByAppendingString:ns];
 图片放在cache目录下
 [directory stringByAppendingPathComponent:fullNamespace]
 最终地址：
 目录+图片名（cachekey的MD5）
// 根据image的key(一般情况下理解为image的url)组合成最终的文件路径
// 上面那个生成的文件路径只是一个文件目录，就跟/cache/images/img1.png和
	cache/images/的区别一样
// defaultCachePathForKey后面会详解
 NSString *cachePathForKey = [self defaultCachePathForKey:key];
 // transform to NSUrl
 NSURL *fileURL = [NSURL fileURLWithPath:cachePathForKey];

 [_fileManager createFileAtPath:cachePathForKey 
 						contents:data attributes:nil];

 // disable iCloud backup
 //iCloud 不备份
 if (self.shouldDisableiCloud) {
     [fileURL setResourceValue:[NSNumber numberWithBool:YES]
     	 				forKey:NSURLIsExcludedFromBackupKey
     	  				 error:nil];
 }


```


### 缓存图片的查找：

#### 根据cachekey 判断这个图片是否缓存过
```
- (NSOperation *)queryDiskCacheForKey:(
NSString *)key done:(SDWebImageQueryCompletedBlock)doneBlock

```

#### 从缓存中查找

```
- (UIImage *)imageFromMemoryCacheForKey:(NSString *)key

UIImage *image = [self imageFromMemoryCacheForKey:key];
if (image) {
    doneBlock(image, SDImageCacheTypeMemory);
    return nil;
}

如果查找到了，直接返回图片，如果没找到请看下一步

```

#### 从磁盘中查找

```
   NSOperation *operation = [NSOperation new];
    dispatch_async(self.ioQueue, ^{
        if (operation.isCancelled) {
            return;
        }

        @autoreleasepool {
            UIImage *diskImage = [self diskImageForKey:key];
            if (diskImage && self.shouldCacheImagesInMemory) {
                NSUInteger cost = 
                			SDCacheCostForImage(diskImage);
                [self.memCache setObject:diskImage forKey:key cost:cost];
            }

            dispatch_async(dispatch_get_main_queue(), ^{
                doneBlock(diskImage, SDImageCacheTypeDisk);
            });
        }
    });

```

在磁盘中查找此图片

```
- (UIImage *)diskImageForKey:(NSString *)key {
    NSData *data = [self diskImageDataBySearchingAllPathsForKey:key];
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

根据key获取到本地磁盘中关于这个图片的数据，注意这里stringByDeletingPathExtension方法以后可以多借鉴

```
- (NSData *)diskImageDataBySearchingAllPathsForKey:(NSString *)key {
    NSString *defaultPath = [self defaultCachePathForKey:key];
    NSData *data = [NSData dataWithContentsOfFile:defaultPath];
    if (data) {
        return data;
    }

    // checking the key with and without the extension
    data = [NSData dataWithContentsOfFile:
    			[defaultPath stringByDeletingPathExtension]];
    if (data) {
        return data;
    }

    NSArray *customPaths = [self.customPaths copy];
    for (NSString *path in customPaths) {
        NSString *filePath = 
        				[self cachePathForKey:key inPath:path];
        NSData *imageData = 
        				[NSData dataWithContentsOfFile:filePath];
        				
        if (imageData) {
            return imageData;
        }

        // checking the key with and without the extension
        imageData = 
        			[NSData dataWithContentsOfFile:
        			[filePath stringByDeletingPathExtension]];
        if (imageData) {
            return imageData;
        }
    }

    return nil;
}

```

根据key，获取这个图片在本地如果存在的话，返回图片的路径，这里的方法和图片下载完成之后图片的存储地址的生成是相同的

```
- (NSString *)defaultCachePathForKey:(NSString *)key {
    return [self cachePathForKey:key inPath:self.diskCachePath];
}

- (NSString *)cachePathForKey:(NSString *)key inPath:(NSString *)path {
    NSString *filename = [self cachedFileNameForKey:key];
    return [path stringByAppendingPathComponent:filename];
}

```

对图片名称进行MD5的方法（可以保留做一个工具方法）

```
- (NSString *)cachedFileNameForKey:(NSString *)key {
    const char *str = [key UTF8String];
    if (str == NULL) {
        str = "";
    }
    
    // 使用了MD5进行加密处理
    // 开辟一个16字节（128位：md5加密出来就是128bit）的空间
    unsigned char r[CC_MD5_DIGEST_LENGTH];
    // 官方封装好的加密方法
    // 把str字符串转换成了32位的16进制数列（这个过程不可逆转） 
    	存储到了r这个空间中
    CC_MD5(str, (CC_LONG)strlen(str), r);
    // 最终生成的文件名就是 "md5码"+".文件类型"
    NSString *filename = 
    				     [NSString stringWithFormat:
    						@"%02x%02x%02x%02x%02x%02x%02x%02x
    						%02x%02x%02x%02x%02x%02x%02x%02x%@",
              r[0], r[1], r[2], r[3], r[4], r[5], r[6], 
              r[7], r[8], r[9], r[10], r[11], r[12], r[13], 
              r[14], r[15], 
              [[key pathExtension] isEqualToString:@""] 
              ? @"" 
              : [NSString stringWithFormat:@".%@", [key 											pathExtension]]];

    return filename;
}

```

计算图片的消耗
 
	cost 被用来计算缓存中所有对象的代价。当内存受限或者所有缓存对象的总代价超过了最大允许的值时，缓存会移除其中的一些对象。
	通常，精确的 cost 应该是对象占用的字节数
 
```
FOUNDATION_STATIC_INLINE NSUInteger SDCacheCostForImage(UIImage *image) {
	//这里我觉得这样写不是很好，如果这样写就更直观了
	// return (height * scale) * (width * scale)
	
    return image.size.height * image.size.width * image.scale * image.scale;
}

```

图片找到之后

```
dispatch_main_sync_safe(^{
  __strong __typeof(weakOperation) strongOperation =
  												 weakOperation;
  if (strongOperation && !strongOperation.isCancelled) {
     completedBlock(image, nil, cacheType, YES, url);
  }
});
 @synchronized (self.runningOperations) {
    [self.runningOperations removeObject:operation];
 }

```
