---
layout: post
title: "打分控件-支持滑动和点击打分"
date: 2015-12-15 14:52:57 +0800
category: Demo
tags: [iOS Objective-C]

---
#### 简介：

	项目开发中通常都会用到打分的控件，所以这里简单的封装了一下，支持点击和滑动打分
	
实现思路：
	利用手势，我是预先在这个打分的view上添加了两层view 一个是灰色星星的view一个是高亮时星星的按钮，然后根据手势获取滑动的距离动态去改变高亮星星的view的frame 以实现打分的效果



#### 具体实现：

通过添加手势(UIPanGestureRecognizer),根据手势的两种状态

```
UIGestureRecognizerStateBegan 手势开始

UIGestureRecognizerStateChanged 手势移动

```
根据手势的状态获取分数

```
- (void)handlePan:(UIPanGestureRecognizer *)recognizer {
    static CGFloat startX = 0;
    CGFloat starScorePercentage = 0;
    
    if (recognizer.state == UIGestureRecognizerStateBegan) {
        startX = [recognizer locationInView:self].x;
        starScorePercentage = startX / self.starWidth;
        
    } else if (recognizer.state == 
    						UIGestureRecognizerStateChanged) 
    {
        CGFloat location = 
        		[recognizer translationInView:self].x + startX;
        starScorePercentage = location / self.starWidth;
    } else {
        return;
    }
    
    CGFloat realScore = self.allowIncompleteStar ? 
    								starScorePercentage : 
  									ceilf(starScorePercentage);
    								
    self.percentage = realScore / self.starCount;
}	

```
self.allowIncompleteStar表示是否允许设置非全星

重写layoutSubviews

```
    CGFloat duration = self.shouldUseAnimation ? 0.5 : 0.0f;
    [UIView animateWithDuration:duration animations:^{
        weakSelf.lightStarView.frame = CGRectMake(0, 0, 		weakSelf.percentage * weakSelf.bounds.size.width, 		weakSelf.bounds.size.height);
        
        NSLog(@"%f%f",
        weakSelf.bounds.size.width,weakSelf.bounds.size.height);
    }];

```
self.shouldUseAnimation 表示是否需要添加动画

#### 传值：
这里可以根据这个方法将当前打分的值，传递给外面
	
```
@property (nonatomic, strong) void (^currentPercentBlock)(CGFloat percent);

```

这样就可以实现一个简单的打分控件了

[代码在这里](https://github.com/LeeWongSnail/ArtRatingView)
