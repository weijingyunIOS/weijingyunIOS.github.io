---
layout: post
title: "Swift Little Tips"
date: 2016-02-15 13:10:05 +0800
comments: true
category:Swift
 
---

```
	本文主要讲述了，我在学习swift过程中遇到的一些问题以及总结出的一些技巧，在这里分享给大家。
	
```

####1、方法的重载

大家都知道OC是不支持方法的重载（存在两个名称相同的方法）的，而swift支持方法的重载，所以在项目中，我们可能会遇到下面这个问题

我的代码

```
import UIKit

class BugViewController: UIViewController
{
    func perform(operation: (Double) -> Double) {
    }

    func perform(operation: (Double, Double) -> Double) {
    }
}

```

错误

```
Method 'perform' with Objective-C selector 'perform: ' conflicts with previous declaration with the same Objective-C selector

```
疑问：

我是在用纯swift写代码，为什么会提示OC的东西？
原因：你的代码中包含了OC或者间接继承了OC的相关对象(我这里是因为控制器继承自UIViewController)
解决方法：

1、去掉和OC相关的部分（删除继承）

```
import UIKit

class BugViewController
{
    func perform(operation: (Double) -> Double) {
    }

    func perform(operation: (Double, Double) -> Double) {
    }
}

```

2、添加@objc

在重载的方法前，加上@nonobjc

```
import UIKit

class BugViewController: UIViewController
{
    func perform(operation: (Double) -> Double) {
    }
    
	@nonobjc
    func perform(operation: (Double, Double) -> Double) {
    }
}


```

3、添加private标志

```
import UIKit

class BugViewController: UIViewController
{
    func perform(operation: (Double) -> Double) {
    }
    
   private func perform(operation: (Double, Double) -> Double) {
    }
}

```
详细请参考[链接](http://stackoverflow.com/questions/29457720/compiler-error-method-with-objective-c-selector-conflicts-with-previous-declara/31500740#31500740)


####2、类方法

实例方法是被类型的某个实例调用的方法。你也可以定义类型本身调用的方法，这种方法就叫做类型方法。声明类的类型方法，在方法的func关键字之前加上关键字class；声明结构体和枚举的类型方法，在方法的func关键字之前加上关键字static

```
注意：
在 Objective-C 里面，你只能为 Objective-C 的类定义类型方法(type-level methods)。在 Swift 中，你可以为所有的类、结构体和枚举定义类型方法：每一个类型方法都被它所支持的类型显式包含。

```
示例：

```
class SomeClass {
  class func someTypeMethod() {
    // type method implementation goes here
  }
}
SomeClass.someTypeMethod()

```


参考文章：

[1、static和class的使用](http://www.topthink.com/topic/8972.html)

####3、Class has no initializers错误

![errorTips](http://img.hoop8.com/attachments/1602/9323165027545.png)

这是我在swift学习中常见的一个错误，
这个错误通常是由于你的这个类中声明的某一个变量，
在声明的时候没有指定这个变量是否是optional，
这时你只需要检查一下，自己声明的变量是否需要添加！或者？

####4、获取类名或者实例的类型

使用类名获取

```
UITableViewCell.self   UITableViewCell.classForCoder()

```
实例对象获取类型

```
   let date = NSDate()
   let name = date.dynamicType

```

[获取对象类型](http://swifter.tips/instance-type/)