---
layout: post
title: "SDWebImage 解析"
date: 2016-01-26 10:48:05 +0800
comments: true
categories: 
---

###目录
- 0、一张图片的下载流程
- 1、[图片的下载](http://leewongsnail.github.io/blog/2016/01/27/sdwebimageyuan-ma-jie-xi-xia-zai/)
- 2、[图片的缓存](http://leewongsnail.github.io/blog/2016/01/27/sdwebimageyuan-ma-jie-xi-huan-cun/)
- 3、分类的解析
- 4、SD常识汇总

###SDWebImage的层次结构：
![icon](http://draveness.me/content/images/2015/04/Untitled-2.png)

接下来，就按照图片下载的流程来分析一下SDWebImage

###图片下载和缓存流程

这一步主要就是跟踪一个图片的下载流程，通过这个流程，我们可以大致的了解SD的图片下载过程中的重要的方法以及处理问题的步骤！

`方法过程`:

```
最外层图片下载 可以添加占位图
- (void)sd_setImageWithURL:(NSURL *)url placeholderImage:(UIImage *)placeholder options:(SDWebImageOptions)options 

```

```
添加一个对于下载进度的监听的回调，下载完成的回调
- (void)sd_setImageWithURL:(NSURL *)url placeholderImage:(UIImage *)placeholder options:(SDWebImageOptions)options progress:(SDWebImageDownloaderProgressBlock)progressBlock completed:(SDWebImageCompletionBlock)completedBlock 

```

```
开始下载（包括查找缓存的过程）
- (id <SDWebImageOperation>)downloadImageWith
					URL:(NSURL *)url
                   options:(SDWebImageOptions)options
                   progress:(SDWebImageDownloaderProgressBlock)progressBlock
                   completed:(SDWebImageCompletionWithFinishedBlock)completedBlock 

```

```
查找图片
- (NSOperation *)queryDiskCacheForKey:(NSString *)key done:(SDWebImageQueryCompletedBlock)doneBlock 
```

```
从缓存SDImageCache中查找图片
- (UIImage *)imageFromMemoryCacheForKey:(NSString *)key 
如果查找到图片
```

```
从磁盘中查找图片
- (UIImage *)diskImageForKey:(NSString *)key

```

```
开始下载
- (id <SDWebImageOperation>)downloadImageWith
						URL:(NSURL *)url 
						options:										(SDWebImageDownloaderOptions)options
						 progress:							(SDWebImageDownloaderProgressBlock)progressBlock
						  completed:(SDWebImageDownloaderCompletedBlock)completedBlock 

```

```
添加下载进度的回调
- (void)addProgressCallback:								(SDWebImageDownloaderProgressBlock)progressBlock 
			  completedBlock:								(SDWebImageDownloaderCompletedBlock)completedBlock
			   forURL:(NSURL *)url
			 createCallback:				
			 (SDWebImageNoParamsBlock)createCallback

```

```
下载的请求
- (id)initWithRequest:(NSURLRequest *)request
              options:(SDWebImageDownloaderOptions)options
             progress:(SDWebImageDownloaderProgressBlock)progressBlock
            completed:(SDWebImageDownloaderCompletedBlock)completedBlock
            cancelled:(SDWebImageNoParamsBlock)cancelBlock 

```
上面是简单的通过对一个图片的下载跟踪过程，整理出来的几个比较重要的方法,图片下载的流程就是-先从缓存中查找看是否已经下载-如果有直接显示，如果没有从磁盘中查找-如果找到现将图片移动到缓存中，在显示图片，如果没有找到开始下载；

在下面对这一步简单描述的各个流程进行简单的细化，过程中可能会存在一些问题，我们能也暂时说不清楚，会用重点的标志标记一下，希望看到这篇文章的同学也可以跟我联系，共同学习；

###1、[图片的下载](http://leewongsnail.github.io/blog/2016/01/27/sdwebimageyuan-ma-jie-xi-xia-zai/)
	
###2、[图片的缓存](http://leewongsnail.github.io/blog/2016/01/27/sdwebimageyuan-ma-jie-xi-huan-cun/)

###3、[缓存管理](http://leewongsnail.github.io/blog/2016/01/27/sdwebimageyuan-ma-jie-xi-huan-cun-guan-li/)

###4、[图片处理](http://leewongsnail.github.io/blog/2016/01/27/sdwebimageyuan-ma-jie-xi-tu-pian-chu-li/)


###5、多线程方面的注意(待写)
