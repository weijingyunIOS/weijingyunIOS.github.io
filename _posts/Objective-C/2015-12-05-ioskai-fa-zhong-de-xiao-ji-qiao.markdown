---
layout: post
title: "iOS开发中的小技巧"
date: 2015-05-05 21:46:14 +0800
comments: true
category:Objective-C
 
---

1、
用 Property() 这个 macro 在编译时检查一个 class 是否包含一个 property，并取到那个 property 的名字（一个 NSString），配合 Core Data 使用非常方便。

```
#define Property(Class, PropertyName) @(((void)(NO && ((void)[Class nilObject].PropertyName, NO)), # PropertyName))
```

2、设置透明的导航栏

```
[self.navigationController.navigationBar setBackgroundImage:[UIImage new] forBarMetrics:UIBarMetricsDefault];
//导航栏底部线清楚
self.navigationController.navigationBar.barStyle = UIBarStyleBlack;
self.navigationController.navigationBar.translucent = YES;[self.navigationController.navigationBar setShadowImage:[UIImage new]];
self.navigationBar.translucent = YES;
```
![icon](https://pic4.zhimg.com/e844607cdaa057885b84142de2e3c297_b.jpg)

3、iOS8 以上 侧滑多按钮

```
- (nullable NSArray
*)tableView:(UITableView *)tableView editActionsForRowAtIndexPath:(NSIndexPath *)indexPath

返回一个UITableViewRowAction数组，每一个"Action"代表一个侧滑删除的Button。这样侧滑每一行Cell可以有更多按钮提供给用户交互。

```

4、设置banner中的图片的时候 一般情况下对图片进行的设置

```
1、aspect fill 让图片填充
2、clip subviews 如果有图片宽度过宽裁剪超出部分（超出部分不会自动裁剪） 
```

5、使用tableviewcell的时候 最好采用注册-使用的方式不要

```
	if( cell == nil )
```
太low


6、最好不要再用#define定义常量

```
原因：1、宏只是进行字符串替换   2  如果作为一个第三方的容易和现有的宏冲突 如果非要用 必须加上 if define

如何定义常量
static NSString const kImageWidth= 80.f
使用这种方式定义
```

7、手动调用一个按钮的点击事件（适合于使用RAC或者block的情况）

```
 [cell.callTeacherBtn sendActionsForControlEvents:UIControlEventTouchUpInside];
 
```
8、控制台输出 想要的数据

![icon](http://ww1.sinaimg.cn/bmiddle/cb8a22eagw1eys7l6nzz5j20gb07ajsw.jpg)

这个略叼啊！！！！

9、tableview 动态计算行高
方法1：传入这个cell对应的模型，然后一次进行计算
方法2：传入模型，根据模型更新约束会改变的控件的约束，刷新界面，获取最下面的空间的最大高度

示例：

```
//在表格cell中 计算出高度
-(CGFloat)rowHeightWithCellModel:(HomeModel *)homeModel
{
    _homeModel=homeModel;
    __weak __typeof(&*self)weakSelf = self;
    //设置标签的高度
    [self.content mas_makeConstraints:^(MASConstraintMaker *make) {
        // weakSelf.contentLabelH  这个会调用下面的懒加载方法
        make.height.mas_equalTo(weakSelf.contentLabelH);
    }];
    
    // 2. 更新约束
    [self layoutIfNeeded];
    
    //3.  视图的最大 Y 值
    CGFloat h= CGRectGetMaxY(self.content.frame);
   
    return h+marginW; //最大的高度+10
}

```

方法2的优化版：

	缓存行高，可以通过在这个cell对应的模型中增加一个属性，在第二次计算的事后，判断这个值是否为0如果不为0，就是用这个值
	
方法3：预估行高
	思路1：使用tableview代理方法estimatedHeightForRowAtIndexPath 给一个预估的行高，这样可以大量的减少行高的计算，但是在实际应用中，如果给出的这个预估的行高和真是的行高差别较大的时候 会出现cell窜动的现象
![icon](http://s6.51cto.com/wyfs02/M01/6E/F4/wKiom1WMuObjSUiVABLXImYoZ_o796.gif)
	因此对于，tableview中cell有不同的样式的时候不要采用这个方法。

iOS8新特性方法：

```
self.tableView.estimatedRowHeight = 50.0f; 
self.tableView.rowHeight = UITableViewAutomaticDimension; 
```
使用这种方法 苹果会帮你把行高都计算了

10、获取CollectionView当前cell的indexPath

注意：仅适用于 一个界面只显示一个cell，但是 [self.collectionView indexPathsForVisibleItems] 返回的有时候个数不只是一个 所以第三种方法是最安全的（前两种方法适用于一般的情况，不适用于二级联动的情况）

方法1：

```
- (void)collectionView:(UICollectionView *)collectionView
  didEndDisplayingCell:(UICollectionViewCell *)cell
    forItemAtIndexPath:(NSIndexPath *)indexPath {
    // 获取当前显示的cell的下标
    NSIndexPath *firstIndexPath = [[self.collectionView indexPathsForVisibleItems] firstObject]; //一定要第一个
    // 赋值给记录当前坐标的变量
    self.currentIndexPath = firstIndexPath;
}
```
方法2：

```
- (void)scrollViewDidEndDecelerating:(UIScrollView *)scrollView {
    // 获取当前显示的cell的下标
    NSIndexPath *firstIndexPath = [[self.collectionView indexPathsForVisibleItems] firstObject];
    // 赋值给记录当前坐标的变量
    self.currentIndexPath = firstIndexPath;
    // 更新底部的数据 
    // ...
}

```

方法3：

```
- (void)scrollViewDidEndDecelerating:(UIScrollView *)scrollView {
    // 将collectionView在控制器view的中心点转化成collectionView上的坐标
    CGPoint pInView = [self.view convertPoint:self.collectionView.center toView:self.collectionView];
    // 获取这一点的indexPath
    NSIndexPath *indexPathNow = [self.collectionView indexPathForItemAtPoint:pInView];
    // 赋值给记录当前坐标的变量
    self.currentIndexPath = indexPathNow;
    // 更新底部的数据 
    // ...
}

```

11、rangeOfString:&rangeOfString:option:
 在平时的使用中大家都比较习惯使用，下面这种方式！
```
	NSString *str = @":/lalalal";

	NSRange range = [str rangeOfString:@":"];

但是，这里的range.lenght = 0;
跟重要的是 len = str.length; 结果len = 0; 

原因：

```
Unicode对于组成有两种形式：合成形式与分解形式。
而NSString的rangeOfString，这个api对此的支持是这样的。rangeOfString，默认不是按照码元来查找的，也就是不是按照literalSearch.虽然它里面包含":"，但是，这两个字符可以合成另一个与其等价的字符，所以就找不到了。
 
```

12、获取launchImage的方法

	launchimage程序启动过程中加载的那张图片，加载完毕就会消失，但有时候我们不希望他那么快的消失，比如需要添加广告页的时候，这时候 我们就需要获取到这张图片，把他作为背景图
	
获取这张图片的方法


1 为不同分辨率的屏幕设置不同的图片名称，使用的时候通过拼接图片名称的方式获取，但是这种方式比较依赖于图片的命名，一旦屏幕分辨率改变了 就需要改动

2、通过下面的代码 从bundle中读取，前提是 图片通过Asset.xcassets设置

```
    NSArray* imagesDict = [[[NSBundle mainBundle] infoDictionary] valueForKey:@"UILaunchImages"];
    for (NSDictionary* dict in imagesDict)
    {
        CGSize imageSize = CGSizeFromString(dict[@"UILaunchImageSize"]);

        if (CGSizeEqualToSize(imageSize, viewSize) && [viewOrientation isEqualToString:dict[@"UILaunchImageOrientation"]])
        {
            launchImage = dict[@"UILaunchImageName"];
        }
    }
   launchImage 就是这张图片的名称
```


13、设置tableviewcell的分割线左对齐

```
if ([[[UIDevice currentDevice] systemVersion] floatValue] >= 7) {

self.tableView.separatorInset = UIEdgeInsetsMake(0, 0, 0, 0);

}
```
查看更多的设置方法：
[分割线左对齐](http://www.skyfox.org/ios7-tableview-separatorinset-ajust.html)


14、去掉tableview中section的headerview粘性

```
- (void)scrollViewDidScroll:(UIScrollView *)scrollView  
{  
    CGFloat sectionHeaderHeight = 40;  
    if (scrollView.contentOffset.y<=sectionHeaderHeight&&scrollView.contentOffset.y>=0) {  
        scrollView.contentInset = UIEdgeInsetsMake(-scrollView.contentOffset.y, 0, 0, 0);  
    }  
    else if (scrollView.contentOffset.y>=sectionHeaderHeight) {  
        scrollView.contentInset = UIEdgeInsetsMake(-sectionHeaderHeight, 0, 0, 0);  
    }  
}

```

15、纯手码布局的好帮手
	使用下面的这个宏，在你做界面布局的时候，有些控件通常要根据屏幕的尺寸设置，使用这个宏就可以以6P为基准，直接拿到你想要的转换之后的数值了
	
	#define SYRealValue(value) ((value)/414.0f*[UIScreen mainScreen].bounds.size.width)
	
	
16、如何隐藏tableview中的某一行

```
- (CGFloat) tableView:(UITableView *)tableView heightForRowAtIndexPath:
(NSIndexPath *)indexPath {
    return indexPath.row == 3 ? 0 : 40;
}

```
	这样未必有效果吧，有时可能会出现：
![图片超出喽](http://upload-images.jianshu.io/upload_images/139317-120fc766b0282440.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

所以还需要另外一句

```
- (void)tableView:(UITableView *)tableView willDisplayCell:(UITableViewCell *)cell forRowAtIndexPath:(NSIndexPath *)indexPath {
    cell.clipsToBounds = YES;
}

```

17、如何改变UITextfield的placeholder的颜色
集成UITextfield重写这个方法

```
- (void) drawPlaceholderInRect:(CGRect)rect {
    [[UIColor blueColor] setFill];
    [self.placeholder drawInRect:rect withFont:self.font lineBreakMode:UILineBreakModeTailTruncation alignment:self.textAlignment];
}

```
也可以使用KVC

```
[textField setValue:[UIColor blueColor] forKeyPath:@"_placeholderLabel.textColor"]

```

18、本来我的statusbar是lightcontent的，结果用UIImagePickerController会导致我的statusbar的样式变成黑色，怎么办？

```
- (void)navigationController:(UINavigationController *)navigationController willShowViewController:(UIViewController *)viewController animated:(BOOL)animated
{
    [[UIApplication sharedApplication] setStatusBarStyle:UIStatusBarStyleLightContent];
}

```

19、如何修改tableviewcell中选中符号的颜色

```
_mTableView.tintColor = [UIColor redColor];

```

20、ScrollView莫名其妙不能在viewController划到顶怎么办?

这个可能是设置了导航栏的背景图片导致的

```
self.automaticallyAdjustsScrollViewInsets = NO;

```

21、怎么点击self.view就让键盘收起,需要添加一个tapGestures么

```
- (void)touchesBegan:(NSSet *)touches withEvent:(UIEvent *)event
{
   [self.view endEditing:YES];
}

```

22、怎么像safari一样滑动的时候隐藏navigationbar?

```
navigationController.hidesBarsOnSwipe = Yes

```

、、、、、、未完待续、、、、、、