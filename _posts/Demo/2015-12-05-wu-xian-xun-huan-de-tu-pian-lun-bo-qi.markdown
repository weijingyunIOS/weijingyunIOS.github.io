---
layout: post
title: "无限循环的图片轮播器"
date: 2015-04-05 19:27:52 +0800
category: Demo
tags: [iOS Objective-C]

---

简介：本篇文章主要是介绍使用UIScrollView创建一个无限循环的图片轮播器，banner在许多应用中都有比较广泛的使用，希望这篇文章会对你有所帮助。

#### 一、界面的搭建
这里的图片轮播期只是简单的一个UIScrollView+UIPageControl

1、界面搭建

```
    self.scrollView = [[UIScrollView alloc] init];
    self.scrollView.showsHorizontalScrollIndicator = NO;
    self.scrollView.showsVerticalScrollIndicator = NO;
    self.scrollView.pagingEnabled = YES;
    self.scrollView.bounces = NO;
    self.scrollView.delegate = self;
    self.scrollView.userInteractionEnabled = YES;
    [self.contentView addSubview:self.scrollView];
    [self.scrollView makeConstraints:^(MASConstraintMaker *make) {
        make.edges.equalTo(self.contentView);
    }];
    
    [self.scrollView setContentOffset:CGPointMake(kScreenW, 0)];
    
    
    //UIPageControl
    self.pageControl = [[UIPageControl alloc] init];
    self.pageControl.pageIndicatorTintColor = [UIColor whiteColor];
    self.pageControl.currentPageIndicatorTintColor = [UIColor colorWithRGBHex:0xF64E4E];
    self.pageControl.backgroundColor = [UIColor clearColor];
    self.pageControl.hidesForSinglePage = YES;
    [self.contentView addSubview:self.pageControl];
    [self.pageControl makeConstraints:^(MASConstraintMaker *make) {
        make.bottom.equalTo(self.scrollView.bottom).offset(-5.);
        make.right.equalTo(self.contentView.right).offset(-11);
        make.height.equalTo(@20);
    }];

    
```
注意：这里为什么要设置contentoffset为一个屏幕宽度的偏移下面会详细介绍

#### 二、数据准备

1、思路：

```
	无限循环的图片轮播器，主要的难点就是，第一张左滑滚动到最后一张，最后一张右滑滚动到第一张。
	这里在数据部分 我们做一个简单的处理，假设你有5张（a b c d e）图片要展示 那么 要显示的数据应该是7个(e a b c d e a) 这样可以使得在图片左滑或者右滑到最后一张的时候可以预览（`这里解释为什么开始的时候要设置一个偏移量`）
```
2、代码处理：

```
	self.banners = [NSMutableArray 										arrayWithArray:banners.banners];
    [self.banners insertObject:banners.banners.lastObject 					atIndex:0];
    [self.banners addObject:banners.banners.firstObject];
```

#### 三、处理滚动
1、在ScrollView上图片添加完毕的时候一个图片轮播器已经基本搭建起来

注意:不要忘记设置contentSize
	
```
    self.scrollView.contentSize = CGSizeMake(kScreenW * self.banners.count, [ArtBannerCell height]);
```
2、定时滚动
添加定时器

方法1、

```
    _timer = [NSTimer scheduledTimerWithTimeInterval:5.f target:self selector:@selector(nextStoryDisplay) userInfo:nil repeats:YES];
    
```

方法2、

```
    self.timer = [NSTimer scheduledTimerWithTimeInterval:2.f target:self selector:@selector(nextImage) userInfo:nil repeats:YES];
    
    [[NSRunLoop mainRunLoop] addTimer:self.timer forMode:NSRunLoopCommonModes];
    
```
区别：方法1 当banner被添加在一个列表上时，滑动列表banner会暂停滚动
	 方法2 定时器被添加到运行循环中，列表滚动不会导致banner暂停
	 
	 
参考:[招聘一个靠谱的iOS](https://github.com/ChenYilong/iOSInterviewQuestions) 02-30题

3、无限轮播
先上代码

```
//定时调用方法
- (void)nextImage {
    [self.scrollView setContentOffset:CGPointMake(self.scrollView.contentOffset.x+kScreenW, 0) animated:YES];
}

//设置临界的时候的滚动
- (void)scrollViewDidScroll:(UIScrollView *)scrollView {
    NSInteger count = self.bannerList.banners.count;
    if ([scrollView isEqual:_scrollView]) {
        CGFloat offSetX = scrollView.contentOffset.x;
        if (offSetX == (count + 1)*kScreenW) {
            _scrollView.contentOffset = CGPointMake(kScreenW, 0);
            _pageControl.currentPage = 0;
        }else if (offSetX == 0){
            _scrollView.contentOffset = CGPointMake(count*kScreenW, 0);
            _pageControl.currentPage = count - 1;
        }else {
            _pageControl.currentPage = offSetX/kScreenW-1;
        }
    }
}
```
看了上面的代码 发现 So Easy ...... 只需要在滚动到第一张或者最后一张的时候设置一下scrollview的Contentoffset就OK了 是不是很简单啊

#### 四、总结
	这只是一个最最基本的banner的实现，实际应用中可能会增加一些其他的元素，这里就不在赘述。希望这篇文章可以帮到你！
	

