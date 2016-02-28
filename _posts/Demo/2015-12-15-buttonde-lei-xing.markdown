---
layout: post
title: "button的类型"
date: 2015-03-15 12:19:34 +0800
categories: Demo
tags: [iOS Objective-C]
categories: 
---
简介：

	在我们平时的项目中，一般都会遇到各种类型的按钮，都有文字或者图片，但是文字和图片的位置都不确定，下面写的是一个非常简单的一个按钮实现按钮的集中显示方式
	
1、可能的显示方式

```
typedef NS_ENUM(NSInteger, ArtButtonStyle) {
    ArtButtonStyleImageTop,
    ArtButtonStyleImageLeft,
    ArtButtonStyleImageRight,
    ArtButtonStyleImageBottom
};

```

2、具体的实现，其实我只是需要在layoutSubviews的时候，重新写一下按钮内部的布局就可以了

(1)图片在顶部

```
if (aStyle == ArtButtonStyleImageTop) {
        
        CGFloat imgTop = (self.frame.size.height
        		 -(self.imageView.frame.size.height 
        		 	+ self.titleLabel.frame.size.height 
        		 	+ kTitleImageMargin))/2;
        			
        CGFloat imgLeft = (self.frame.size.width 
        				- self.imageView.frame.size.width)/2.;
        				
        CGFloat imgHeight = self.imageView.frame.size.height;
        CGFloat imgWidth = self.imageView.frame.size.width;
        
        self.imageView.frame = CGRectMake(imgLeft, imgTop, 											imgWidth, imgHeight);
        
        CGFloat lblTop = CGRectGetMaxY(self.imageView.frame) 
        									+ kTitleImageMargin;
        CGFloat lblLeft = (self.frame.size.width 
        				- self.titleLabel.frame.size.width)/2;
        				
        [self.titleLabel sizeToFit];
        self.titleLabel.frame = CGRectMake(lblLeft, lblTop, 							self.titleLabel.frame.size.width, 							self.titleLabel.frame.size.height);
        
    }
```

(2)图片在底部

```
if (aStyle == ArtButtonStyleImageBottom) {
        
      CGFloat lblTop = (self.frame.size.height - 						(self.imageView.frame.size.height
      						 + self.titleLabel.frame.size.height 							 + kTitleImageMargin))/2;
      						 
      CGFloat lblLeft = (self.frame.size.width - 							self.titleLabel.frame.size.width)/2;
      
        [self.titleLabel sizeToFit];
        
        self.titleLabel.frame = CGRectMake(lblLeft, lblTop, 							self.titleLabel.frame.size.width, 							self.titleLabel.frame.size.height);
        
        CGFloat imgTop = CGRectGetMaxY(self.titleLabel.frame)
        								 + kTitleImageMargin;
        								 
        CGFloat imgLeft = (self.frame.size.width
        				 - self.imageView.frame.size.width)/2.;
        
        CGFloat imgHeight = self.imageView.frame.size.height;
        CGFloat imgWidth = self.imageView.frame.size.width;
    
        self.imageView.frame = CGRectMake(imgLeft, imgTop, 											imgWidth, imgHeight);
    }

```

(3)图片在右边

```
if (aStyle == ArtButtonStyleImageRight) {
       //交换图片和文字的位置
        CGRect lblRect = self.titleLabel.frame;
        lblRect.origin.x = self.imageView.frame.origin.x;
        self.titleLabel.frame = lblRect;
        
        CGRect imgRect = self.imageView.frame;
        imgRect.origin.x =
         CGRectGetMaxX(self.titleLabel.frame) 							+kTitleImageMargin;
        self.imageView.frame = imgRect;
      
}

```

(4)图片在左边

```
if (aStyle == ArtButtonStyleImageLeft) {
        //默认情况
        [self setTitleEdgeInsets:UIEdgeInsetsMake(0, 									kTitleImageMargin, 0, 0)];
        
    }

```

看一下最终的效果：
![效果图](http://img.hoop8.com/attachments/1512/0633412352730.png)

[代码在这里](https://github.com/LeeWongSnail/FancyButton)