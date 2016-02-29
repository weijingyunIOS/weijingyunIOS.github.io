---
layout: post
title: "面试笔记"
date: 2015-07-23 14:47:05 +0800
comments: true
category: Tools

---
##面试题知识点总结
###1、枚举的定义
	NS_ENUM和NS_OPTIONS本质是一样的，仅仅从字面上来区分其用途。NS_ENUM是通用情况，NS_OPTIONS一般用来定义具有位移操作或特点的情况（bitmask)。
	typedef NS_ENUM(NSInteger, UIViewAnimationTransition) {  
    UIViewAnimationTransitionNone,//默认从0开始  
    UIViewAnimationTransitionFlipFromLeft,  
    UIViewAnimationTransitionFlipFromRight,  
    UIViewAnimationTransitionCurlUp,  
    UIViewAnimationTransitionCurlDown,  
	};  
  
	typedef NS_OPTIONS(NSUInteger, UIViewAutoresizing) {  
    	UIViewAutoresizingNone                 = 0,  
    	UIViewAutoresizingFlexibleLeftMargin   = 1 << 0,  
    	UIViewAutoresizingFlexibleWidth        = 1 << 1,  
    	UIViewAutoresizingFlexibleRightMargin  = 1 << 2,  
    	UIViewAutoresizingFlexibleTopMargin    = 1 << 3,  
    	UIViewAutoresizingFlexibleHeight       = 1 << 4,  
    	UIViewAutoresizingFlexibleBottomMargin = 1 << 5  
	};
###2、方法名中间的连词
	//错误，不要使用"and"来连接参数
	- (int)runModalForDirectory:(NSString *)path andFile:	(NSString *)name andTypes:(NSArray *)fileTypes;
	//错误，不要使用"and"来阐明有多个参数
	- (instancetype)initWithName:(CGFloat)width andAge:	(CGFloat)height;
	//正确，使用"and"来表示两个相对独立的操作
	- (BOOL)openFile:(NSString *)fullPath withApplication:	(NSString *)appName andDeactivate:(BOOL)flag;
###3、编程规范的问题
	在-和(void)之间应该有一个空格
	enum 中驼峰命名法和下划线命名法混用错误：枚举类型的命名规则和函数的	命名规则相同：命名时使用驼峰命名法，勿使用下划线命名法。
	enum 左括号前加一个空格，或者将左括号换到下一行
	enum 右括号后加一个空格
	UserModel :NSObject 应为UserModel : NSObject，也就是:右侧少	了一个空格。
	@interface 与 @property 属性声明中间应当间隔一行。
	两个方法定义之间不需要换行，有时为了区分方法的功能也可间隔一行，但示	例代码中间隔了两行。
	-(id)initUserModelWithUserName: (NSString*)name withAge:	(int)age;方法中方法名与参数之间多了空格。而且 - 与 (id) 之间少了	空格。
	-(id)initUserModelWithUserName: (NSString*)name withAge:	(int)age;方法中方法名与参数之间多了空格：(NSString*)name 前多了	空格。
	-(id)initUserModelWithUserName: (NSString*)name withAge:	(int)age; 方法中 (NSString*)name,应为 (NSString *)name，少	了空格。
###4、什么情况使用 weak 关键字，相比 assign 有什么不同？

	什么情况使用 weak 关键字？

	在 ARC 中,在有可能出现循环引用的时候,往往要通过让其中一端使用 weak 	来解决,比如: delegate 代理属性

	自身已经对它进行一次强引用,没有必要再强引用一次,此时也会使用 weak,	自定义 IBOutlet 控件属性一般也使用 weak；当然，也可以使用	strong。在下文也有论述：《IBOutlet连出来的视图属性为什么可以被设	置成weak?》

	不同点：
	weak 此特质表明该属性定义了一种非拥有关系 (nonowning 	relationship)。为这种属性设置新值时，设置方法既不保留新值，也不释	放旧值。此特质同assign类似， 然而在属性所指的对象遭到摧毁时，属性值	也会清空(nil out)。 而 assign 的“设置方法”只会执行针对纯量类	型” (scalar type，例如 CGFloat 或 NSlnteger 等)的简赋值操	。
	assigin 可以用非 OC 对象,而 weak 必须用于 OC 对象
###5、怎么用 copy 关键字？
	1、block使用copy
		block 使用 copy 是从 MRC 遗留下来的“传统”,在 MRC 中,方法内部的 block 是在栈区的,使用 copy 可以把它放到堆区.在 ARC 中写不写都行对于 block 使用 copy 还是 strong 效果是一样的
	使用copy的意义：
		编译器自动对 block 进行了 copy 操作。如果不写 copy ，该类的调用者有可能会忘记或者根本不知道“编译器会自动对 block 进行了 copy 操作”，他们有可能会在调用之前自行拷贝属性值。这种操作多余而低效。
	
![Mou icon](https://camo.githubusercontent.com/8a5fa34435801cc4c2715d8880f3abd45be6a6c5/687474703a2f2f692e696d6775722e636f6d2f566c564b6c384c2e706e67)
```
	下面做下解释： copy 此特质所表达的所属关系与 strong 类似。然而设置方法并不保留新值，而是将其“拷贝” (copy)。 当属性类型为 NSString 时，经常用此特质来保护其封装性，因为传递给设置方法的新值有可能指向一个 NSMutableString 类的实例。这个类是 NSString 的子类，表示一种可修改其值的字符串，此时若是不拷贝字符串，那么设置完属性之后，字符串的值就可能会在对象不知情的情况下遭人更改。所以，这时就要拷贝一份“不可变” (immutable)的字符串，确保对象中的字符串值不会无意间变动。只要实现属性所用的对象是“可变的” (mutable)，就应该在设置新属性值时拷贝一份。
	用 @property 声明 NSString、NSArray、NSDictionary 经常使用 copy 关键字，是因为他们有对应的可变类型：NSMutableString、NSMutableArray、NSMutableDictionary，他们之间可能进行赋值操作，为确保对象中的字符串值不会无意间变动，应该在设置新属性值时拷贝一份。
```
###6、这个写法会出什么问题： @property (copy) NSMutableArray *array;
	两个问题：1、添加,删除,修改数组内的元素的时候,程序会因为找不到对应的方法而崩溃.因为 copy 就是复制一个不可变 NSArray 的对象；2、使用了 atomic 属性会严重影响性能 ；
	1、NSMutableArray *array = [NSMutableArray 								arrayWithObjects:@1,@2,nil];
		self.mutableArray = array;
		[self.mutableArray removeObjectAtIndex:0];
		执行会崩溃
	2、该属性使用了同步锁，会在创建时生成一些额外的代码用于帮助编写多线程程序，这会带来性能问题，通过声明 nonatomic 可以节省这些虽然很小但是不必要额外开销
	
###7、@property 的本质是什么？ivar、getter、setter 是如何生成并添加到这个类中的
	@property 的本质是什么？
	@property = ivar + getter + setter;
	下面解释下：
	“属性” (property)有两大概念：ivar（实例变量）、存取方法（access method ＝ getter + setter）。
	ivar、getter、setter 是如何生成并添加到这个类中的?
	“自动合成”( autosynthesis)

	完成属性定义后，编译器会自动编写访问这些属性所需的方法，此过程叫做“自动合成”(autosynthesis)。需要强调的是，这个过程由编译 器在编译期执行，所以编辑器里看不到这些“合成方法”(synthesized method)的源代码。除了生成方法代码 getter、setter 之外，编译器还要自动向类中添加适当类型的实例变量，并且在属性名前面加下划线，以此作为实例变量的名字。在前例中，会生成两个实例变量，其名称分别为 _firstName 与 _lastName。也可以在类的实现代码里通过 @synthesize 语法来指定实例变量的名字.
	@implementation Person
	@synthesize firstName = _myFirstName;
	@synthesize lastName = _myLastName;
	@end
	我为了搞清属性是怎么实现的,曾经反编译过相关的代码,他大致生成了五个东西

	OBJC_IVAR_$类名$属性名称 ：该属性的“偏移量” (offset)，这个偏移	量是“硬编码” (hardcode)，表示该变量距离存放对象的内存区域的起始地	址有多远。
	setter 与 getter 方法对应的实现函数
	ivar_list ：成员变量列表
	method_list ：方法列表
	prop_list ：属性列表
	也就是说我们每次在增加一个属性,系统都会在 ivar_list 中添加一个成员变量的描述,在 method_list 中增加 setter 与 getter 方法的描述,在属性列表中增加一个属性的描述,然后计算该属性在对象中的偏移量,然后给出 setter 与 getter 方法对应的实现,在 setter 方法中从偏移量的位置开始赋值,在 getter 方法中从偏移量开始取值,为了能够读取正确字节数,系统对象偏移量的指针类型进行了类型强转.

###8 @protocol 和 category 中如何使用 @property
	1、在 protocol 中使用 property 只会生成 setter 和 getter 方法声明,我们使用属性的目的,是希望遵守我协议的对象能实现该属性
	2、category 使用 @property 也是只会生成 setter 和 getter 方法的声明,如果我们真的需要给 category 增加属性的实现,需要借助于运行时的两个函数：

	objc_setAssociatedObject
	objc_getAssociatedObject
	
	
###10  @property中有哪些属性关键字？ @property 后面可以有哪些修饰符？
	1、原子性--- nonatomic 特质
	2、读/写权限---readwrite(读写)、readonly (只读)
	3、内存管理语义---assign、strong、 weak、unsafe_unretained、copy
	4、方法名---getter=<name> 、setter=<name>

			getter=<name>的样式：

      @property (nonatomic, getter=isOn) BOOL on;
      
      setter=<name>一般用在特殊的情境下，比如：
      	在数据反序列化、转模型的过程中，服务器返回的字段如果以 init 开头，所以你需要定义一个 init 开头的属性，但默认生成的 setter 与 getter 方法也会以 init 开头，而编译器会把所有以 init 开头的方法当成初始化方法，而初始化方法只能返回 self 类型，因此编译器会报错。
      	避免方法：
      	@property(nonatomic, strong, getter=p_initBy, setter=setP_initBy:)NSString *initBy;
 
###11、weak属性需要在dealloc中置nil么？
	不需要
	在ARC环境无论是强指针还是弱指针都无需在 dealloc 设置为 nil ， ARC 会自动帮我们处理
	在属性所指的对象遭到摧毁时，属性值也会清空(nil out)。
	
###12、 @synthesize和@dynamic分别有什么作用？
	@property有两个对应的词，一个是 @synthesize，一个是 @dynamic。如果 @synthesize和 @dynamic都没写，那么默认的就是@syntheszie var = _var;
	@synthesize 的语义是如果你没有手动实现 setter 方法和 getter 方法，那么编译器会自动为你加上这两个方法。
	@dynamic 告诉编译器：属性的 setter 与 getter 方法由用户自己实现，不自动生成。（当然对于 readonly 的属性只需提供 getter 即可）。假如一个属性被声明为 @dynamic var，然后你没有提供 @setter方法和 @getter 方法，编译的时候没问题，但是当程序运行到 instance.var = someVar，由于缺 setter 方法会导致程序崩溃；或者当运行到 someVar = var 时，由于缺 getter 方法同样会导致崩溃。编译时没问题，运行时才执行相应的方法，这就是所谓的动态绑定。

###13、ARC下，不显式指定任何属性关键字时，默认的关键字都有哪些
	1、对应基本数据类型默认关键字是

	atomic,readwrite,assign

	2、对于普通的 Objective-C 对象

	atomic,readwrite,strong

###14、用@property声明的NSString（或NSArray，NSDictionary）经常使用copy关键字，为什么？如果改用strong关键字，可能造成什么问题
	1、因为父类指针可以指向子类对象,使用 copy 的目的是为了让本对象的属性不受外界影响,使用 copy 无论给我传入是一个可变对象还是不可对象,我本身持有的就是一个不可变的副本.
	2、如果我们使用是 strong ,那么这个属性就有可能指向一个可变对象,如果这个可变对象在外部被修改了,那么会影响该属性.
	对非集合类对象的copy操作：
	[immutableObject copy] // 浅复制
	[immutableObject mutableCopy] //深复制
	[mutableObject copy] //深复制
	[mutableObject mutableCopy] //深复制
	集合类对象的copy与mutableCopy：
	[immutableObject copy] // 浅复制
	[immutableObject mutableCopy] //单层深复制
	[mutableObject copy] //单层深复制
	[mutableObject mutableCopy] //单层深复制
	在集合类对象中，对 immutable 对象进行 copy，是指针复制， mutableCopy 是内容复制；对 mutable 对象进行 copy 和 mutableCopy 都是内容复制。但是：集合对象的内容复制仅限于对象本身，对象元素仍然是指针复制
	
###15、@synthesize合成实例变量的规则是什么？假如property名为foo，存在一个名为_foo的实例变量，那么还会自动合成新变量么？
.h

	@interface CYLPerson : NSObject 
	@property NSString *firstName; 
	@property NSString *lastName; 
	@end
在上例中，会生成两个实例变量，其名称分别为 _firstName 与 _lastName。也可以在类的实现代码里通过 @synthesize 语法来指定实例变量的名字:
.m

	@implementation CYLPerson 
	@synthesize firstName = _myFirstName; 
	@synthesize lastName = _myLastName; 
	@end
上述语法会将生成的实例变量命名为 _myFirstName 与 _myLastName ，而不再使用默认的名字

@synthesize 合成实例变量的规则:

	如果指定了成员变量的名称,会生成一个指定的名称的成员变量,
	如果这个成员已经存在了就不再生成了.
	如果是 @synthesize foo; 还会生成一个名称为foo的成员变量，也就是说：如果没有指定成员变量的名称会自动生成一个属性同名的成员变量
	如果是 @synthesize foo = _foo; 就不会生成成员变量了.
	
假如 property 名为 foo，存在一个名为 _foo 的实例变量，那么还会自动合成新变量么？

	不会，
![icon](https://camo.githubusercontent.com/8e11101c9fe0b3defc7fbd144c0dca9fdf0471d0/687474703a2f2f692e696d6775722e636f6d2f743238676534572e706e67)

###16、在有了自动合成属性实例变量之后，@synthesize还有哪些使用场景？

	不会autosynthesis的场景：
		同时重写了 setter 和 getter 时
		重写了只读属性的 getter 时
		使用了 @dynamic 时
		在 @protocol 中定义的所有属性
		在 category 中定义的所有属性
		重载的属性
			当你在子类中重载了父类中的属性，你必须 使用 @synthesize 			来手动合成ivar。
			在子类中重写父类的属性时，编译的时候会提示问题！！！
![icon](https://camo.githubusercontent.com/a569a90281598d8cc74156fe5f0e3a6ddbf8fc6b/687474703a2f2f692e696d6775722e636f6d2f6641454748496f2e706e67)

	当你同时重写了 setter 和 getter 时，系统就不会生成 ivar（实例变量/成员变量）。这时候有两种选择
	要么如第14行：手动创建 ivar
	要么如第17行：使用@synthesize foo = _foo; ，关联 @property 与 ivar。
###17、objc中向一个nil对象发送消息将会发生什么
	在 Objective-C 中向 nil 发送消息是完全有效的——只是在运行时不会有任何作用:
	1、如果一个方法返回值是一个对象，那么发送给nil的消息将返回0(nil)
	2、如果方法返回值为指针类型，其指针大小为小于或者等于sizeof(void*)，float，double，long double 或者 long long 的整型标量，发送给 nil 的消息将返回0
	3、如果方法返回值为结构体,发送给 nil 的消息将返回0。结构体中各个字段的值将都是0。
	4、如果方法的返回值不是上述提到的几种情况，那么发送给 nil 的消息的返回值将是未定义的。
	原因：
	objc是动态语言，每个方法在运行时会被动态转为消息发送，即：objc_msgSend(receiver, selector)。
	
	objc在向一个对象发送消息时，runtime库会根据对象的isa指针找到该对象实际所属的类，然后在该类中的方法列表以及其父类方法列表中寻找方法运行，然后在发送消息的时候，objc_msgSend方法不会返回值，所谓的返回内容都是具体调用时执行的。 那么，回到本题，如果向一个nil对象发送消息，首先在寻找对象的isa指针时就是0地址返回了，所以不会出现任何错误。

###18、objc中向一个对象发送消息[obj foo]和objc_msgSend()函数之间有什么关系？
	该方法编译之后就是objc_msgSend()函数调用
	[obj foo];在objc动态编译时，会被转意为：objc_msgSend(obj, 	@selector(foo));。
		int main(int argc, char * argv[]) {
    	@autoreleasepool {
        	CYLTest *test = [[CYLTest alloc] init];
        	[test performSelector:(@selector(iOSinit))];
        	return 0;
    	}
	}
	具体实现：
![icon](https://camo.githubusercontent.com/bf346edc21ab4b3387906602249b357d250ab1c2/687474703a2f2f692e696d6775722e636f6d2f6541483559576e2e706e67)

###12、 什么时候会报unrecognized selector的异常？
	当调用该对象上某个方法,而该对象上没有实现这个方法的时候， 可以通过“消息转发”进行解决。
	objc是动态语言，每个方法在运行时会被动态转为消息发送，即：objc_msgSend(receiver, selector)。
	objc在向一个对象发送消息时，runtime库会根据对象的isa指针找到该对象实际所属的类，然后在该类中的方法列表以及其父类方法列表中寻找方法运行，如果，在最顶层的父类中依然找不到相应的方法时，程序在运行时会挂掉并抛出异常unrecognized selector sent to XXX 。但是在这之前，objc的运行时会给出三次拯救程序崩溃的机会：
1、Method resolution

	objc运行时会调用+resolveInstanceMethod:或者 +resolveClassMethod:，让你有机会提供一个函数实现。如果你添加了函数，那运行时系统就会重新启动一次消息发送的过程，否则 ，运行时就会移到下一步，消息转发（Message Forwarding）。

2、Fast forwarding

	如果目标对象实现了-forwardingTargetForSelector:，Runtime 这时就会调用这个方法，给你把这个消息转发给其他对象的机会。 只要这个方法返回的不是nil和self，整个消息发送的过程就会被重启，当然发送的对象会变成你返回的那个对象。否则，就会继续Normal Fowarding。 这里叫Fast，只是为了区别下一步的转发机制。因为这一步不会创建任何新的对象，但下一步转发会创建一个NSInvocation对象，所以相对更快点。

3、Normal forwarding

	这一步是Runtime最后一次给你挽救的机会。首先它会发送-methodSignatureForSelector:消息获得函数的参数和返回值类型。如果-methodSignatureForSelector:返回nil，Runtime则会发出-doesNotRecognizeSelector:消息，程序这时也就挂掉了。如果返回了一个函数签名，Runtime就会创建一个NSInvocation对象并发送-forwardInvocation:消息给目标对象。

###13、一个objc对象如何进行内存布局？（考虑有父类的情况）
	所有父类的成员变量和自己的成员变量都会存放在该对象所对应的存储空间中.
	每一个对象内部都有一个isa指针,指向他的类对象,类对象中存放着本对象的
		对象方法列表（对象能够接收的消息列表，保存在它所对应的类对象中）
		成员变量的列表,
		属性列表,
	它内部也有一个isa指针指向元对象(meta class),元对象内部存放的是类方法列表,类对象内部还有一个superclass的指针,指向他的父类对象
![icon](https://camo.githubusercontent.com/cdc02fffae7a70aa00cbb6c0f3675d00728cdaad/687474703a2f2f692e696d6775722e636f6d2f7736747a46787a2e706e67)

###14、一个objc对象的isa的指针指向什么？有什么作用？
	指向他的类对象,从而可以找到对象上的方法
###15、下面的代码输出什么？
```
@implementation Son : Father
    - (id)init
    {
        self = [super init];
        if (self) {
            NSLog(@"%@", NSStringFromClass([self class]));
            NSLog(@"%@", NSStringFromClass([super class]));
        }
        return self;
    }
    @end
```
	都输出 Son
	
	这个题目主要是考察关于 Objective-C 中对 self 和 super 的理解。
	我们都知道：self 是类的隐藏参数，指向当前调用方法的这个类的实例。那 super 呢？

	很多人会想当然的认为“ super 和 self 类似，应该是指向父类的指针吧！”。这是很普遍的一个误区。其实 super 是一个 Magic Keyword， 它本质是一个编译器标示符，和 self 是指向的同一个消息接受者！他们两个的不同点在于：super 会告诉编译器，调用 class 这个方法时，要去父类的方法，而不是本类里的。

	上面的例子不管调用[self class]还是[super class]，接受消息的对象都是当前 Son ＊xxx 这个对象。

	当使用 self 调用方法时，会从当前类的方法列表中开始找，如果没有，就从父类中再找；而当使用 super 时，则从父类的方法列表中开始找。然后调用父类的这个方法。

	这也就是为什么说“不推荐在 init 方法中使用点语法”，如果想访问实例变量 iVar 应该使用下划线（ _iVar ），而非点语法（ self.iVar ）

###16、runtime如何通过selector找到对应的IMP地址？（分别考虑类方法和实例方法
	每一个类对象中都一个方法列表,方法列表中记录着方法的名称,方法实现,以及参数类型,其实selector本质就是方法名称,通过这个方法名称就可以在方法列表中找到对应的方法实现.
###17、. 使用runtime Associate方法关联的对象，需要在主对象dealloc的时候释放么
	无论在MRC下还是ARC下均不需要。
```
// 对象的内存销毁时间表
// http://weibo.com/luohanchenyilong/ (微博@iOS程序犭袁)
// https://github.com/ChenYilong
// 根据 WWDC 2011, Session 322 (36分22秒)中发布的内存销毁时间表 

 1. 调用 -release ：引用计数变为零
     * 对象正在被销毁，生命周期即将结束.
     * 不能再有新的 __weak 弱引用， 否则将指向 nil.
     * 调用 [self dealloc] 
 2. 父类 调用 -dealloc
     * 继承关系中最底层的父类 在调用 -dealloc
     * 如果是 MRC 代码 则会手动释放实例变量们（iVars）
     * 继承关系中每一层的父类 都在调用 -dealloc
 3. NSObject 调 -dealloc
     * 只做一件事：调用 Objective-C runtime 中的 object_dispose() 方法
 4. 调用 object_dispose()
     * 为 C++ 的实例变量们（iVars）调用 destructors 
     * 为 ARC 状态下的 实例变量们（iVars） 调用 -release 
     * 解除所有使用 runtime Associate方法关联的对象
     * 解除所有 __weak 引用
     * 调用 free()
```
	
###18、 objc中的类方法和实例方法有什么本质区别和联系？

	类方法：

	类方法是属于类对象的
	类方法只能通过类对象调用
	类方法中的self是类对象
	类方法可以调用其他的类方法
	类方法中不能访问成员变量
	类方法中不定直接调用对象方法

	实例方法：
	实例方法是属于实例对象的
	实例方法只能通过实例对象调用
	实例方法中的self是实例对象
	实例方法中可以访问成员变量
	实例方法中直接调用实例方法
	实例方法中也可以调用类方法(通过类名)
###9  runtime 如何实现 weak 属性
	weak属性的特点：
	weak 此特质表明该属性定义了一种“非拥有关系” (nonowning relationship)。为这种属性设置新值时，设置方法既不保留新值，也不释放旧值。此特质同 assign 类似， 然而在属性所指的对象遭到摧毁时，属性值也会清空(nil out)。