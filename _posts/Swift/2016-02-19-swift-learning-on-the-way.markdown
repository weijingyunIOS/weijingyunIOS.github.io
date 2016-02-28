---
layout: post
title: "Swift Learning On The Way"
date: 2016-02-19 14:54:19 +0800
comments: true
categories: 
---

2015年已经过去了，iOS也做了有一段时间了。这两年Swift的势头越来越猛，所以，我觉得是时候开始系统的去学习和练习使用Swift了。

之前看了斯坦福大学白胡子的视频，对于Swift也算稍微了解了一点，近期打算看一些关于Swift相关的东西，看到了[Allen朝辉](http://weibo.com/ttarticle/p/show?id=2309403942494873235448)的《自学 iOS - 三十天三十个 Swift 项目》感触很大，所以 我们打算在未来的一段时间里，学习Swift的同时坚持写一些小的Demo练手，希望自己坚持下去。

关于基础的语法，我觉得看一下中文版的[Swift开发指南](http://wiki.jikexueyuan.com/project/swift/?hmsr=www_index_wiki)就好,下面将会附上，以后这段时间 每天的输出！

[Demo的代码放在这里](https://github.com/LeeWongSnail/Swift-On-The-Way)

####Project01 - SimpleStopWatch

一个可以暂停和开始和重置的计时器

![example](http://img.hoop8.com/attachments/1602/9201790664455.gif)


重点练习了Autolayout布局，以及一个小的tips，在一个文件内使用performSelector调用一个私有方法的时候回报野指针的错误，可以通过在前面加`@objc`的方式解决这个问题

####Project02 - CustomFont

自定义字体

![gitexample](http://img.hoop8.com/attachments/1602/2701790664455.gif)

iOS 中有些字体是需要版权的因此可能需要购买，我这里使用的字体是Allen朝辉Demo中提供的ttf文件，并在plist文件中添加：

Fonts provided by application 属性 表明具体的字体

tips：获取当前显示字体的名称

```
    for family in UIFont.familyNames() {
            for font in UIFont.fontNamesForFamilyName(family){
                orginFont = font
            }
        }

```

####Project03 - Play Local Video

播放本地文件

![play local video](http://img.hoop8.com/attachments/1602/0581790515984.gif)

需要导入

import AVKit

import AVFoundation

`练习一下协议的使用方式`



####Project04 - SnapChatMenu
联系scrollView的布局
可以参考[里脊的笔记](http://adad184.com/2015/12/01/scrollview-under-autolayout/)

![snapChatMenu](http://img.hoop8.com/attachments/1602/2283165027545.gif)

将一个控制器的view添加到另一个控制器中

```
  let midVC = MidViewController(nibName: 
        					"MidViewController", bundle: nil)
  self.addChildViewController(midVC)
  self.scrollView.addSubview(midVC.view)
  midVC.didMoveToParentViewController(self)
  var midFrame = midVC.view.frame
  midFrame.origin.x = screenSize.width
  midVC.view.frame = midFrame

```
涉及到拍摄界面的先关内容，这里还看得不太懂，以后再讲……

关于隐藏状态栏，我在使用

```
UIApplication.sharedApplication().statusBarHidden = true

```
方法设置的时候没有起作用，所以改用下面的方法

```
    override func prefersStatusBarHidden() -> Bool {
        return true
    }

```

####Project05 - Carousel Effect
CollectionView简单使用

![icon](http://img.hoop8.com/attachments/1602/3643165027545.gif)

了解UICollectionView的使用方法，学会使用extension在代理和数据源中的使用，学习使用Visual Effect View。

[UIVisualEffectView 学习](http://www.cocoachina.com/ios/20150604/11987.html)

还有一个问题是在swift中如何区分类方法和对象方法以及使用static修饰的方法有什么特性，具体可以在我的[这篇博客](http://leewongsnail.github.io/blog/2016/02/15/swift-little-tips/)看到


####Project06 - Get My Location
定位功能的简单使用

![get my Location](http://img.hoop8.com/attachments/1602/4513165027545.gif)

可以仔细的查看一下addressDict中的内容。
设置定位使用时间

```
　iOS 8对定位进行了一些修改，其中包括定位授权的方法，CLLocationManager增加了下面的两个方法：
　　（1）始终允许访问位置信息
　　- (void)requestAlwaysAuthorization;
　　（2）使用应用程序期间允许访问位置数据
　　- (void)requestWhenInUseAuthorization;

```

####Project07 - Pull To Refresh

简单的下拉刷新功能

![Pull To Refresh](http://img.hoop8.com/attachments/1602/4783165027545.gif)

tableview这里使用的是纯代码 NavigationBar使用SB设置

熟悉使用extentsion

了解如何设置NavigationBar的一些属性，背景颜色、文字颜色、透明度

注意swift在闭包中使用变量的时候，需要添加self.

`闭包中如何避免循环引用，在后面需要多注意`

####Project08 - Random Color Gradient

就是练习了一下渐变色的使用（CAGradientLayer）

![RandomColorGradient](http://img.hoop8.com/attachments/1602/0533165027545.gif)

```
//init layer
gradientLayer.frame = self.view.bounds
self.view.layer.addSublayer(gradientLayer)
        
//setting direction
gradientLayer.startPoint = CGPointZero
gradientLayer.endPoint = CGPointMake(1, 1)
        
//设置颜色组
for _ in 1...6 {
    let r = CGFloat(drand48())
    let g = CGFloat(drand48())
    let b = CGFloat(drand48())
     colors.append(UIColor(red:r, green: g, blue: b, alpha: 
     	1.0).CGColor)
            
    locs.append(Double(random()%10) / 10.0)
}

gradientLayer.colors = colors
        
gradientLayer.locations = locs

```


####Project09 - ImageScroller
简单的查看大图，可以放大图片，左右滑动

![Image Scroller](http://img.hoop8.com/attachments/1602/7213165027545.gif)

`注意`

```
imageView在初始化的时候 如果想让imageview根据image的大小自己去调整imageview的frame 必须使用

var imageView = UIImageView(image: UIImage(named: "steve"))

进行初始化
如果使用的是：
let image = UIImage(named: "steve")
imageView.image = image
则没有效果

```

####Project10 - SplashVideo
使用一段视频作为背景

![Splash Video Background](http://img.hoop8.com/attachments/1602/6331790443611.gif)

这个demo 涉及到类的集成，以及类中变量的访问控制

修饰类中的变量或者方法 可以使用public private internal 三种

```
Public：可以访问自己模块或应用中源文件里的任何实体，别人也可以访问引入该模块中源文件里的所有实体。通常情况下，某个接口或Framework是可以被任何人使用时，你可以将其设置为public级别。

Internal：可以访问自己模块或应用中源文件里的任何实体，但是别人不能访问该模块中源文件里的实体。通常情况下，某个接口或Framework作为内部结构使用时，你可以将其设置为internal级别。

Private：只能在当前源文件中使用的实体，称为私有实体。使用private级别，可以用作隐藏某些功能的实现细节。

```

`多注意各个属性的设置`

[访问控制参考文档](http://www.devtalking.com/articles/swift-access-control/)


####Porject - 10 - Clear Table View
![clear](http://img.hoop8.com/attachments/1602/0973412352730.gif)

主要就是仿照clear APP 实现一个颜色的渐变


####Project - 11 - Login Animation

一个简单的登录界面的动画

![Login Animation](http://img.hoop8.com/attachments/1602/4181790443611.gif)

这里两个Textfield的动画是直接改变约束做的，如果想要有这样的动画效果，切记不可设置这两个控件距离左边的边距

这里学习使用了Spring动画，可以设置一个速率和一个阻尼系数

```
dampingRatio：阻尼系数，范围为 0.0 ~ 1.0，数值越小，弹簧振动的越厉害，Spring 的效果越明显

velocity：表示速度，数值越大移动的越快。值为 1.0 时，这个速度为 1 秒钟之内走完整个动画距离的速度。更大或更小的值会让 view 刚到达终点时的速度更大或更小

```
[Spring动画参考](http://www.jianshu.com/p/9ed68b2b44d4)


####Project - 12 - AnimatedTableViewCell

实现了tableview内容加载的一个动画

![AnimatedTableViewCell](http://img.hoop8.com/attachments/1602/0971790516245.gif)

复习了 Spring动画的使用

实现的基本思路就是：先将所有第一次要显示的cell下移一段距离，然后在使用动画上移实现动画效果

用SB实现界面间的跳转 给Segue设置一个标示符，然后调用

	performSegueWithIdentifier("pushtoSecond", sender: nil)



####Project - 13 - Twetter Splash

一个模仿推特界面的一个动画

![Twetter Splash](http://img.hoop8.com/attachments/1602/6731790516245.gif)

使用的是关键帧动画

[关于关键帧的学习参考](http://www.devtalking.com/articles/uiview-keyframe-animation/)