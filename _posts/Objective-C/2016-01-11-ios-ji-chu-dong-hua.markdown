---
layout: post
title: "iOS 基础动画"
date: 2014-12-11 18:18:10 +0800
comments: true
category: Objective-C
 
---

###1、CALayer 图层介绍
Layer常用属性：
下面通过一个简单的Demo来看一下常用的属性:

![layer的常用属性](http://img.hoop8.com/attachments/1601/950757074960.png)


```
    //设置阴影
    image.layer.shadowColor = [UIColor blackColor].CGColor;
    //设置阴影的偏移
    image.layer.shadowOffset = CGSizeMake(10, 10);
    //设置不透明度
    image.layer.shadowOpacity = 0.5;
    //设置圆角
    image.layer.cornerRadius = 10;
    //必须设置 强制内部子控件支持圆角效果，少了这个设置，UIImageView将没有圆角效果
    //设置之后将没有阴影效果
    image.layer.masksToBounds = YES;
    
    //设置边框
    image.layer.borderWidth = 5.;
    image.layer.borderColor = [UIColor grayColor].CGColor;
    
    //旋转
    //x轴方向缩小为原来的0.5倍
    image.layer.transform = CATransform3DMakeScale(0.5, 1, 0);

```

###2、图层的创建
1、图层的创建

```
    CALayer *layer = [CALayer layer];
    layer.bounds = CGRectMake(0, 0, 100, 100);
    layer.position = CGPointMake(100, 100);
    //锚点：指明哪一个点在position的位置
    //x,y均在0-1的范围内 （0，0）为原点 （1，1）为右下角
    layer.anchorPoint = CGPointMake(1, 1);
    layer.backgroundColor = [UIColor redColor].CGColor;
    [self.view.layer addSublayer:layer];

```

`注意区分position和anchorPoint`

2、自定义图层的方法

```
1、间子类继承与CALayer,实现drawInContext:方法
2、在控制器中设置代理，实现代理方法回执图层

```

###3、CAAnimation
####1、基础动画 CABasicAnimation
	
主要包括三种：形变 位置变化 旋转 都是通过设置keyPath实现的

```
    CABasicAnimation *anim = [CABasicAnimation animation];
    
//    anim.keyPath = @"bounds";  //形变
//    anim.toValue = [NSValue valueWithCGRect:CGRectMake(0, 0, 50, 50)];
    
//    anim.keyPath = @"position"; //位置变化
//    anim.toValue = [NSValue valueWithCGPoint:CGPointMake(300, 300)];  //终止点的位置
//    anim.byValue = [NSValue valueWithCGPoint:CGPointMake(100, 100)];    //相对移动的位置
    
    anim.keyPath = @"transform"; //旋转
    //围绕x,y轴方向旋转
    anim.toValue = [NSValue valueWithCATransform3D:CATransform3DMakeRotation(M_PI_4, 1, 1, 0)];
    
    anim.duration = 2.0f;
    anim.removedOnCompletion = NO;
    anim.fillMode = @"forwards";
    
    [self.layer addAnimation:anim forKey:nil];

```
####2、关键帧动画 CAKeyframeAnimation

可以为动画设置一个路径，让某一个对象按这个路径去运动，可以设置动画的执行节奏

```
    CAKeyframeAnimation *anim = [CAKeyframeAnimation animationWithKeyPath:@"position"];
    anim.removedOnCompletion = NO;
    //保持最新的状态
    anim.fillMode = kCAFillModeForwards;
    anim.duration = 2.f;
    CGMutablePathRef path = CGPathCreateMutable();
//    CGPathAddRect(path, NULL, CGRectMake(0, 0, 100, 100));
    CGPathAddEllipseInRect(path, NULL, CGRectMake(0, 0, 100, 100));
    
    anim.path = path;
    //设置动画的执行节奏
    anim.timingFunction = [CAMediaTimingFunction functionWithName:kCAMediaTimingFunctionEaseOut];
    
    [self.layer addAnimation:anim forKey:nil];

```

####3、转场动画 CATransition
1、view的转场动画

```
    CATransition *transition = [CATransition animation];
    transition.duration = 2.f;
    transition.timingFunction = [CAMediaTimingFunction functionWithName:kCAMediaTimingFunctionEaseOut];
    
    transition.type = @"push";
    transition.subtype = kCATransitionFromTop;
    
    [_animationView exchangeSubviewAtIndex:0 withSubviewAtIndex:1];
    
    [_animationView.layer addAnimation:transition forKey:@"myAnimation"];
```

2、控制器的专场动画

```
    CATransition *transition = [CATransition animation];
    transition.duration = 2.f;
    transition.timingFunction = [CAMediaTimingFunction functionWithName:kCAMediaTimingFunctionEaseOut];
    
    transition.type = @"rippleEffect";
    transition.subtype = kCATransitionFromTop;
    
    [self.navigationController.view.layer addAnimation:transition forKey:@"navAnimation"];
    
    DetailViewController *detailVc = [[DetailViewController alloc] init];
    [self.navigationController showViewController:detailVc sender:nil];

```
![icon](http://img.hoop8.com/attachments/1601/5941760466397.gif)

支持的过渡效果和过渡方向

```
/* 过渡效果
 fade     //交叉淡化过渡(不支持过渡方向) kCATransitionFade
 push     //新视图把旧视图推出去  kCATransitionPush
 moveIn   //新视图移到旧视图上面   kCATransitionMoveIn
 reveal   //将旧视图移开,显示下面的新视图  kCATransitionReveal
 cube     //立方体翻滚效果
 oglFlip  //上下左右翻转效果
 suckEffect   //收缩效果，如一块布被抽走(不支持过渡方向)
 rippleEffect //滴水效果(不支持过渡方向)
 pageCurl     //向上翻页效果
 pageUnCurl   //向下翻页效果
 cameraIrisHollowOpen  //相机镜头打开效果(不支持过渡方向)
 cameraIrisHollowClose //相机镜头关上效果(不支持过渡方向)
*/
   
/* 过渡方向
 kCATransitionFromRight
 kCATransitionFromLeft
 kCATransitionFromBottom

```
`使用的时候一定要注意，有些事苹果私有的API，小心被拒`


####4、动画组 CAAnimationGroup
将几个动画放到一个动画组里，这些动画会依次执行

```
    CATransition *transition = [CATransition animation];
    transition.duration = 2.f;
    transition.timingFunction = [CAMediaTimingFunction functionWithName:kCAMediaTimingFunctionEaseOut];
    
    transition.type = @"push";
    transition.subtype = kCATransitionFromTop;
    
    
    CATransition *transition1 = [CATransition animation];
    transition1.duration = 2.f;
    transition1.timingFunction = [CAMediaTimingFunction functionWithName:kCAMediaTimingFunctionEaseOut];
    
    transition1.type = @"moveIn";
    transition1.subtype = kCATransitionFromTop;
    
    CAAnimationGroup *group = [CAAnimationGroup animation];
    group.animations = @[transition,transition1];
    
    
    [_animationView exchangeSubviewAtIndex:0 withSubviewAtIndex:1];
    
    [_animationView.layer addAnimation:group forKey:@"myAnimation"];

```
