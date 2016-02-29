---
layout: post
title: "面试题——ChenYilong-总结"
date: 2015-08-13 11:27:30 +0800
comments: true
category: Tools

---

###1、_objc_msgForward函数是做什么的，直接调用它将会发生什么？
	_objc_msgForward是 IMP 类型，用于消息转发的：当向一个对象发送一条消息，但它并没有实现的时候，_objc_msgForward会尝试做消息转发。
	
	在上篇中的《objc中向一个对象发送消息[obj foo]和objc_msgSend()函数之间有什么关系？》曾提到objc_msgSend在“消息传递”中的作用。在“消息传递”过程中，objc_msgSend的动作比较清晰：首先在 Class 中的缓存查找 IMP （没缓存则初始化缓存），如果没找到，则向父类的 Class 查找。如果一直查找到根类仍旧没有实现，则用_objc_msgForward函数指针代替 IMP 。最后，执行这个 IMP 。
	
	为了展示消息转发的具体动作，这里尝试向一个对象发送一条错误的消息，并查看一下_objc_msgForward是如何进行转发的。
![icon](https://camo.githubusercontent.com/c06953c83cf1fd56eff670b88e4c3e0cc739c62d/687474703a2f2f692e696d6775722e636f6d2f556a626d5676422e706e67)

操作过程：
![icon](https://camo.githubusercontent.com/c5d6a506acdf37eefcb090e3f7911d85186623f4/687474703a2f2f692e696d6775722e636f6d2f414145527a31542e706e67)
	结合《NSObject官方文档》，排除掉 NSObject 做的事，剩下的就是_objc_msgForward消息转发做的几件事：

	调用resolveInstanceMethod:方法 (或 resolveClassMethod:)。允许用户在此时为该 Class 动态添加实现。如果有实现了，则调用并返回YES，那么重新开始objc_msgSend流程。这一次对象会响应这个选择器，一般是因为它已经调用过class_addMethod。如果仍没实现，继续下面的动作。

	调用forwardingTargetForSelector:方法，尝试找到一个能响应该消息的对象。如果获取到，则直接把消息转发给它，返回非 nil 对象。否则返回 nil ，继续下面的动作。注意，这里不要返回 self ，否则会形成死循环。

	调用methodSignatureForSelector:方法，尝试获得一个方法签名。如果获取不到，则直接调用doesNotRecognizeSelector抛出异常。如果能获取，则返回非nil：创建一个 NSlnvocation 并传给forwardInvocation:。

	调用forwardInvocation:方法，将第3步获取到的方法签名包装成 Invocation 传入，如何处理就在这里面了，并返回非ni。

	调用doesNotRecognizeSelector: ，默认的实现是抛出异常。如果第3步没能获得一个方法签名，执行该步骤。
	
下面回答下第二个问题“直接_objc_msgForward调用它将会发生什么？”

_objc_msgForward 方法解析

-- | _objc_msgForward参数 | 类型
:----------- | :-----------: | -----------:
1         | 所属对象        | id类型
2         | 方法名         | SEL类型
3         | 可变参数        | 可变参数类型

为了直观，我们可以通过如下方式定义一个 IMP类型 ：

	typedef void (*voidIMP)(id, SEL, ...)
	
	一旦调用_objc_msgForward，将跳过查找 IMP 的过程，直接触发“消息转发”，
	如果调用了_objc_msgForward，即使这个对象确实已经实现了这个方法，你也会告诉objc_msgSend：
	“我没有在这个对象里找到这个方法的实现”
	有哪些场景需要直接调用_objc_msgForward？最常见的场景是：你想获取某方法所对应的NSInvocation对象。举例说明：
	JSPatch （Github 链接）就是直接调用_objc_msgForward来实现其核心功能的：

	JSPatch（） 以小巧的体积做到了让JS调用/替换任意OC方法，让iOS APP具备热更新的能力、
	作者的博文《JSPatch实现原理详解》详细记录了实现原理，有兴趣可以看下。
	
	
###2、runtime如何实现weak变量的自动置nil？
	runtime 对注册的类， 会进行布局，对于 weak 对象会放入一个 hash 表中。 用 weak 指向的对象内存地址作为 key，当此对象的引用计数为0的时候会 dealloc，假如 weak 指向的对象内存地址是a，那么就会以a为键， 在这个 weak 表中搜索，找到所有以a为键的 weak 对象，从而设置为 nil。
	我们可以设计一个函数（伪代码）来表示上述机制：
	objc_storeWeak(&a, b)函数：
	objc_storeWeak函数把第二个参数--赋值对象（b）的内存地址作为键值key，将第一个参数--weak修饰的属性变量（a）的内存地址（&a）作为value，注册到 weak 表中。如果第二个参数（b）为0（nil），那么把变量（a）的内存地址（&a）从weak表中删除，
	你可以把objc_storeWeak(&a, b)理解为：objc_storeWeak(value, key)，并且当key变nil，将value置nil。
	
注意：

	在b非nil时，a和b指向同一个内存地址，在b变nil时，a变nil。此时向a发送消息不会崩溃：在Objective-C中向nil发送消息是安全的。

	而如果a是由assign修饰的，则： 在b非nil时，a和b指向同一个内存地址，在b变nil时，a还是指向该内存地址，变野指针。此时向a发送消息极易崩溃。
	原因：第一种情况a适用weak修饰 在b被置为nil的时候会遍历所有指向这块内存地址的指针 将这些指针置为ni 
		第二种情况a适用assign修饰，在b被置为nil的时候 不会将a指向的对象置为nil，所以默认a仍然指向原来的地址 但是这块地址被释放或者重新利用之后 现存的数据类型和原来的可能不一样了，所以可能出现野指针的问题
	
###3、objc_initWeak和objc_destroyWeak解释
 	通过objc_initWeak函数初始化“附有weak修饰符的变量（obj1）”，在变量作用域结束时通过objc_destoryWeak函数释放该变量（obj1）。
 	
方法的内部实现

```
objc_initWeak函数的实现是这样的：在将“附有weak修饰符的变量（obj1）”初始化为0（nil）后，会将“赋值对象”（obj）作为参数，调用objc_storeWeak函数。

	obj1 = 0；
	obj_storeWeak(&obj1, obj)
```

weak 修饰的指针默认值是 nil （在Objective-C中向nil发送消息是安全的）
然后obj_destroyWeak函数将0（nil）作为参数，调用objc_storeWeak函数
objc_storeWeak(&obj1, 0);

```
id obj1;
obj1 = 0;
objc_storeWeak(&obj1, obj);
/* ... obj的引用计数变为0，被置nil ... */
objc_storeWeak(&obj1, 0);
```
objc_storeWeak函数把第二个参数--赋值对象（obj）的内存地址作为键值，将第一个参数--weak修饰的属性变量（obj1）的内存地址注册到 weak 表中。如果第二个参数（obj）为0（nil），那么把变量（obj1）的地址从weak表中删除

###4、能否向编译后得到的类中增加实例变量？能否向运行时创建的类中添加实例变量？为什么？
	不能向编译后得到的类中增加实例变量；
能向运行时创建的类中添加实例变量；
解释下：

	因为编译后的类已经注册在 runtime 中，类结构体中的 objc_ivar_list 实例变量的链表 和 instance_size 实例变量的内存大小已经确定，同时runtime 会调用 class_setIvarLayout 或 class_setWeakIvarLayout 来处理 strong weak 引用。所以不能向存在的类中添加实例变量；

	运行时创建的类是可以添加实例变量，调用 class_addIvar 函数。但是得在调用 objc_allocateClassPair 之后，objc_registerClassPair 之前，原因同上。

###5、runloop和线程有什么关系
1、主线程的run loop默认是启动的。
	iOS的应用程序里面，程序启动后会有一个如下的main()函数

	int main(int argc, char * argv[]) {
	@autoreleasepool {
    	return UIApplicationMain(argc, argv, nil, 	NSStringFromClass([AppDelegate class]));
	}
	}
	重点是UIApplicationMain()函数，这个方法会为main thread设置一个NSRunLoop对象，这就解释了：为什么我们的应用可以在无人操作的时候休息，需要让它干活的时候又能立马响应。
	对其它线程来说，run loop默认是没有启动的，如果你需要更多的线程交互则可以手动配置和启动，如果线程只是去执行一个长时间的已确定的任务则不需要。
	在任何一个 Cocoa 程序的线程中，都可以通过以下代码来获取到当前线程的 run loop 。
	NSRunLoop *runloop = [NSRunLoop currentRunLoop];
[《Objective-C之run loop详解》](http://blog.csdn.net/wzzvictory/article/details/9237973)

###6、不手动指定autoreleasepool的前提下，一个autorealese对象在什么时刻释放？（比如在一个vc的viewDidLoad中创建）
	分两种情况：手动干预释放时机、系统自动去释放。

	手动干预释放时机--指定autoreleasepool 就是所谓的：当前作用域大括号结束时释放。
	系统自动去释放--不手动指定autoreleasepool

	Autorelease对象出了作用域之后，会被添加到最近一次创建的自动释放池中，并会在当前的 runloop 迭代结束时释放。
![icon](https://camo.githubusercontent.com/56f8ea718f47679e7771d247d8f6f820de2e0ab5/687474703a2f2f6936312e74696e797069632e636f6d2f32386b6f6477702e6a7067)

	我们都是知道： 所有 autorelease 的对象，在出了作用域之后，会被自动添加到最近创建的自动释放池中。

	但是如果每次都放进应用程序的 main.m 中的 autoreleasepool 中，迟早有被撑满的一刻。这个过程中必定有一个释放的动作。何时？

	在一次完整的运行循环结束之前，会被销毁。

	那什么时间会创建自动释放池？运行循环检测到事件并启动后，就会创建自动释放池。

	子线程的 runloop 默认是不工作，无法主动创建，必须手动创建。
	@autoreleasepool 当自动释放池被销毁或者耗尽时，会向自动释放池中的所有对象发送 release 消息，释放自动释放池中的所有对象。
	如果在一个vc的viewDidLoad中创建一个 Autorelease对象，那么该对象会在 viewDidAppear 方法执行前就被销毁了。
	如果在一个vc的viewDidLoad中创建一个 Autorelease对象，那么该对象会在 viewDidAppear 方法执行前就被销毁了。
	
[《黑幕背后的Autorelease》](http://blog.sunnyxx.com/2014/10/15/behind-autorelease/)

###7、BAD_ACCESS在什么情况下出现？
	访问了野指针，比如对一个已经释放的对象执行了release、访问已经释放对象的成员变量或者发消息。 死循环
	
###8、苹果是如何实现autoreleasepool的？
	autoreleasepool 以一个队列数组的形式实现,主要通过下列三个函数完成.

	objc_autoreleasepoolPush
	objc_autoreleasepoolPop
	objc_autorelease
	看函数名就可以知道，对 autorelease 分别执行 push，和 pop 操作。销毁对象时执行release操作。
![icon](https://camo.githubusercontent.com/1e77679169328e5128722b3268bf9a488fc00ae2/687474703a2f2f6936302e74696e797069632e636f6d2f31356d666a31312e6a7067)


###9、 如何用GCD同步若干个异步调用？（如根据若干个url异步加载多张图片，然后在都下载完成后合成一张整图
	dispatch_queue_t queue = dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0);
	dispatch_group_t group = dispatch_group_create();
	dispatch_group_async(group, queue, ^{ /*加载图片1 */ });
	dispatch_group_async(group, queue, ^{ /*加载图片2 */ });
	dispatch_group_async(group, queue, ^{ /*加载图片3 */ }); 
	dispatch_group_notify(group, dispatch_get_main_queue(), 	^{
        	// 合并图片
	});

###10、apple用什么方式实现对一个对象的KVO？
	当你观察一个对象时，一个新的类会被动态创建。这个类继承自该对象的原本的类，并重写了被观察属性的 setter 方法。重写的 setter 方法会负责在调用原 setter 方法之前和之后，通知所有观察对象：值的更改。最后通过 isa 混写（isa-swizzling） 把这个对象的 isa 指针 ( isa 指针告诉 Runtime 系统这个对象的类是什么 ) 指向这个新创建的子类，对象就神奇的变成了新创建的子类的实例

![icon](https://camo.githubusercontent.com/9517b0d78961b5f32cf3392b99964f2e1f79fb35/687474703a2f2f6936322e74696e797069632e636f6d2f7379353775722e6a7067)


