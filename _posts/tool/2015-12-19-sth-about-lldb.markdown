---
layout: post
title: "Sth About LLDB"
date: 2015-12-19 06:19:41 +0800
comments: true
category: Tools

--- 

#### 前言：
  
	今天在微博上看到[小笨狼](http://www.jianshu.com/users/1f93e3b1f3da/latest_articles)分享的一片文章，其中比较详细的介绍了LLDB的使用，顿时感觉原来LLDB控制体可以做那么多事情，下面是我挑选的几个，开发中比较常用的命令
	

####1、基本语法
首先还是先看一下最近本的语法，这会让我们会更容易的去理解和记住下面的命令

```
<command> [<subcommand> [<subcommand>...]] 
<action> [-options [option-value]] 
[argument [argument...]]

```	
参数解析：
1、`<command>`和<subcommand>:是LLDB调试命令的名称，命令和子命令是按照层级结构排列的
一个命令对象为跟随他的子对象创建一个上下文，子命令又为子命令创建一个上下文

2、`<action>`:执行命令的操作

3、`<option>`：命令选项

4、`<argument>`:命令参数

5、`[]`:表示命令是可选的，可有可无

exmp:

```
breakpoint set - n main

```
1、`command`:breakpoint 表示断点命令

2、`断点命令`:set 表示设置断点

3、`-n`:表示根据方法name设置断点

4、`argument`: main 表示方法名为main

####2、唯一匹配原则

LLDB的命令遵循唯一匹配原则：假如根据前n个字母已经能唯一匹配到某个命令，则只写前n个字母等效于写下完整的命令。
e.g:

```
breakpoint set -n main
br s -n main

```
下面一行的命令可以唯一匹配到上一行的命令，所以二者是等效的

`~/.lldbinit`

LLDB有了一个启动时加载的文件~/.lldbinit，每次启动都会加载。所以一些初始化的事儿，我们可以写入~/.lldbinit中，比如给命令定义别名等。但是由于这时候程序还没有真正运行，也有部分操作无法在里面玩，比如设置断点。


####3、LLDB常用命令

#####1、expression
语法格式

```
expression <cmd-options> -- <expr>
```
作用：执行一个表达式，并将表达式的返回值输出

e.g 1：执行一个表达式

```
 expression -- self.view.backgroundColor = [UIColor redColor]

```

e.g 2:输出返回值

```
 (lldb) expression -- self.view
    (UIView *) $1 = 0x00007fe322c18a10

```
`expression 可以简写做e`

#####2、p & print & call
其实这三个方法都可以看做是expression的别名，只是不再向expression那样可以设置命令选项（一般情况下，我们是不需要去设置命令选项的）

print:打印出某个东西，可以是变量或者是表达式

p:可以看做是print的缩写

call:调用某个方法

e.g:

```
(lldb) expression -- self.view
(UIView *) $5 = 0x00007fb2a40344a0
(lldb) p self.view
(UIView *) $6 = 0x00007fb2a40344a0
(lldb) print self.view
(UIView *) $7 = 0x00007fb2a40344a0
(lldb) call self.view
(UIView *) $8 = 0x00007fb2a40344a0
(lldb) e self.view
(UIView *) $9 = 0x00007fb2a40344a0

```

#####3、PO
通常我们在控制台打印的时候，一般都是变量值或者是对象
如果我们直接

```
expression -- self.view
(UIView *) $13 = 0x00007fb2a40344a0
```
显然结果并不是我们想要的，所以，我们需要添加命令选项：

```
(lldb) expression -O -- self.view

<UIView: 0x7fb2a40344a0; frame = (0 0; 375 667); autoresize = W+H; layer = <CALayer: 0x7fb2a4018c80>>

```
这里 为了方便使用，LLDB为expression定义了一个别名 po,因此：

```
(lldb) po self.view

<UIView: 0x7fb2a40344a0; frame = (0 0; 375 667); autoresize = W+H; layer = <CALayer: 0x7fb2a4018c80>>

```
也能达到上面一样的效果

#####4、thread backtrace & bt
作用：打印线程的堆栈

```
thread backtrace [-c <count>] [-s <frame-index>] [-e <boolean>]
```
thread backtrace 后面的都是命令选项

`-c`: 设置打印堆栈帧的帧数（frame）
`-s`: 设置从哪一帧开始打印
`-e`: 是否显示额外的回溯

`实际上，我们在平时的使用中一般都不会添加这些命令选项`

e.g: crash happens

```
(lldb) thread backtrace
* thread #1: tid = 0xdd42, 0x000000010afb380b libobjc.A.dylib`objc_msgSend + 11, queue = 'com.apple.main-thread', stop reason = EXC_BAD_ACCESS (code=EXC_I386_GPFLT)
    frame #0: 0x000000010afb380b libobjc.A.dylib`objc_msgSend + 11
  * frame #1: 0x000000010aa9f75e TLLDB`-[ViewController viewDidLoad](self=0x00007fa270e1f440, _cmd="viewDidLoad") + 174 at ViewController.m:23
    frame #2: 0x000000010ba67f98 UIKit`-[UIViewController loadViewIfRequired] + 1198
    frame #3: 0x000000010ba682e7 UIKit`-[UIViewController view] + 27
    frame #4: 0x000000010b93eab0 UIKit`-[UIWindow addRootViewControllerViewIfPossible] + 61
    frame #5: 0x000000010b93f199 UIKit`-[UIWindow _setHidden:forced:] + 282
    frame #6: 0x000000010b950c2e UIKit`-[UIWindow makeKeyAndVisible] + 42

```
这里，我们可以比较清晰的看到 crash 触发的位置，就可以有针对性的进行检查

`backtrace = bt` 可以使用简单的bt来代替backtrace

#####5、thread return
作用： return 返回，表示不再执行断点下面的代码，直接返回，`可以自己定义要返回的值`

e.g: 
![icon](http://upload-images.jianshu.io/upload_images/1122433-cf22e45902233a0c.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

如果我们不想执行断点后面的代码，同时还想让这个方法的返回值为NO

```
(lldb) thread return NO

```

#####6、c & n & s & finish

![icon](http://upload-images.jianshu.io/upload_images/1122433-17ba978ac411af3d.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

其实这四个按钮都可以用下面的命令来代替：

1、`c`/`continue`/`thread continue`: 这三个命令的效果等同于图片中的第一个按钮，表示继续运行

2、`n`/`next`/`thread step-over`:等同于第二个按钮，表示单步执行

3、`s`/`step`/`thread step-in`:等同于第三个按钮，表示进入方法内部

4、`finish`/`step-out`:等同于第四个按钮，表示直接走完当前方法，返回到上一层

######7、breakpoint 
1、breakpoint set 设置断点

```
这个方法的功能比较强大，可以根据方法名设置断点，可以给指定文件中的指定方法设置断点(方法不存在，无法设置断点)，给指定文件的某一行设置断点、设置条件断点、设置单次执行断点。由于这些在实际的开发中用的比较少，所以在这里不再详细说明（需要看的可以在本文最后查看引用的文章）

```
2、breakpoint command add 给断点添加命令

3、breakpoint command list 查看某个断点已有的命令

4、breakpoint command delete 删除断点

5、breakpoint list 查看一ing设置断点的位置

6、breakpoint enable/disable 断点暂时失效/生效

7、breakpoint delete 彻底删除所有断点（也可以删除指定行）


######8、watchpoint 
`breakpoint`有一个孪生兄弟`watchpoint`。如果说`breakpoint`是对方法生效的断点，`watchpoint`就是对地址生效的断点

基本用法和breakpoint 类似，这里不再详细说明

######9、target

对于target这个命令，我们用得最多的可能就是target modules lookup。由于LLDB给target modules取了个别名image，所以这个命令我们又可以写成image lookup


1、image lookup --address 查找这个地址具体对应的文件位置（具体到行）
当我们遇到崩溃（不是停留在main函数中的崩溃）我们可以通过这个方法获取到这个控制器的哪一行出现问题
e.g : crash happens

```
2015-12-17 14:51:06.301 TLLDB[25086:246169] *** Terminating app due to uncaught exception 'NSRangeException', reason: '*** -[__NSArray0 objectAtIndex:]: index 1 beyond bounds for empty NSArray'
*** First throw call stack:
(
    0   CoreFoundation                      0x000000010accde65 __exceptionPreprocess + 165
    1   libobjc.A.dylib                     0x000000010a746deb objc_exception_throw + 48
    2   CoreFoundation                      0x000000010ac7c395 -[__NSArray0 objectAtIndex:] + 101
    3   TLLDB                               0x000000010a1c3e36 -[ViewController viewDidLoad] + 86
    4   UIKit                               0x000000010b210f98 -[UIViewController loadViewIfRequired] + 1198
    5   UIKit                               0x000000010b2112e7 -[UIViewController view] + 27


```

```
(lldb) image lookup -a 0x000000010a1c3e36
      Address: TLLDB[0x0000000100000e36] (TLLDB.__TEXT.__text + 246)
      Summary: TLLDB`-[ViewController viewDidLoad] + 86 at ViewController.m:32

```
准确定位，崩溃出现的方法名，及在文件中的行号

2、image lookup --name

e.g : 

```
某个第三方SDK用了一个我们项目里原有的第三方库，库里面对NSDictionary添加了category。也就是有2个class对NSDictionary添加了名字相同的category，项目中调用自己的category的地方实际走到了第三方SDK里面去了。最大的问题是，这2个同名category方法行为并不一致，导致出现bug
现在问题来了，怎么寻找到底是哪个第三方SDK？方法完全包在.a里面

```

```

(lldb) image lookup -n dictionaryWithXMLString:
2 matches found in /Users/jiangliancheng/Library/Developer/Xcode/DerivedData/VideoIphone-aivsnqmlwjhxapdlvmdmrubbdxpq/Build/Products/Debug-iphoneos/BaiduIphoneVideo.app/BaiduIphoneVideo:
        Address: BaiduIphoneVideo[0x00533a7c] (BaiduIphoneVideo.__TEXT.__text + 5414908)
        Summary: BaiduIphoneVideo`+[NSDictionary(SAPIXmlDictionary) dictionaryWithXMLString:] at XmlDictionary.m
         Module: file = "/Users/jiangliancheng/Library/Developer/Xcode/DerivedData/VideoIphone-aivsnqmlwjhxapdlvmdmrubbdxpq/Build/Products/Debug-iphoneos/BaiduIphoneVideo.app/BaiduIphoneVideo", arch = "armv7"
    CompileUnit: id = {0x00000000}, file = "/Users/jiangliancheng/Development/Work/iOS_ShareLib/SharedLib/Srvcs/BDPassport4iOS/BDPassport4iOS/SAPI/Extensive/ThirdParty/XMLDictionary/XmlDictionary.m", language = "Objective-C"
       Function: id = {0x23500000756}, name = "+[NSDictionary(SAPIXmlDictionary) dictionaryWithXMLString:]", range = [0x005a6a7c-0x005a6b02)
       FuncType: id = {0x23500000756}, decl = XmlDictionary.m:189, clang_type = "NSDictionary *(NSString *)"
         Blocks: id = {0x23500000756}, range = [0x005a6a7c-0x005a6b02)
      LineEntry: [0x005a6a7c-0x005a6a98): /Users/jiangliancheng/Development/Work/iOS_ShareLib/SharedLib/Srvcs/BDPassport4iOS/BDPassport4iOS/SAPI/Extensive/ThirdParty/XMLDictionary/XmlDictionary.m
         Symbol: id = {0x0000f2d5}, range = [0x005a6a7c-0x005a6b04), name="+[NSDictionary(SAPIXmlDictionary) dictionaryWithXMLString:]"
       Variable: id = {0x23500000771}, name = "self", type = "Class", location =  [sp+32], decl = 
       Variable: id = {0x2350000077e}, name = "_cmd", type = "SEL", location =  [sp+28], decl = 
       Variable: id = {0x2350000078b}, name = "string", type = "NSString *", location =  [sp+24], decl = XmlDictionary.m:189
       Variable: id = {0x23500000799}, name = "data", type = "NSData *", location =  [sp+20], decl = XmlDictionary.m:192
        Address: BaiduIphoneVideo[0x012ee160] (BaiduIphoneVideo.__TEXT.__text + 19810016)
        Summary: BaiduIphoneVideo`+[NSDictionary(XMLDictionary) dictionaryWithXMLString:] at XMLDictionary.m
         Module: file = "/Users/jiangliancheng/Library/Developer/Xcode/DerivedData/VideoIphone-aivsnqmlwjhxapdlvmdmrubbdxpq/Build/Products/Debug-iphoneos/BaiduIphoneVideo.app/BaiduIphoneVideo", arch = "armv7"
    CompileUnit: id = {0x00000000}, file = "/Users/wingle/Workspace/qqlive4iphone/iphone_4.0_fabu_20150601/Common_Proj/mobileTAD/VIDEO/Library/Third Party/XMLDictionary/XMLDictionary.m", language = "Objective-C"
       Function: id = {0x79900000b02}, name = "+[NSDictionary(XMLDictionary) dictionaryWithXMLString:]", range = [0x01361160-0x0136119a)
       FuncType: id = {0x79900000b02}, decl = XMLDictionary.m:325, clang_type = "NSDictionary *(NSString *)"
         Blocks: id = {0x79900000b02}, range = [0x01361160-0x0136119a)
      LineEntry: [0x01361160-0x01361164): /Users/wingle/Workspace/qqlive4iphone/iphone_4.0_fabu_20150601/Common_Proj/mobileTAD/VIDEO/Library/Third Party/XMLDictionary/XMLDictionary.m
         Symbol: id = {0x0003a1e9}, range = [0x01361160-0x0136119c), name="+[NSDictionary(XMLDictionary) dictionaryWithXMLString:]"
       Variable: id = {0x79900000b1e}, name = "self", type = "Class", location =  r0, decl = 
       Variable: id = {0x79900000b2c}, name = "_cmd", type = "SEL", location =  r1, decl = 
       Variable: id = {0x79900000b3a}, name = "string", type = "NSString *", location =  r2, decl = XMLDictionary.m:325
       Variable: id = {0x79900000b4a}, name = "data", type = "NSData *", location =  r2, decl = XMLDictionary.m:327
```
找到file:

```
CompileUnit: id = {0x00000000}, file = "/Users/jiangliancheng/Development/Work/iOS_ShareLib/SharedLib/Srvcs/BDPassport4iOS/BDPassport4iOS/SAPI/Extensive/ThirdParty/XMLDictionary/XmlDictionary.m", language = "Objective-C"
CompileUnit: id = {0x00000000}, file = "/Users/wingle/Workspace/qqlive4iphone/iphone_4.0_fabu_20150601/Common_Proj/mobileTAD/VIDEO/Library/Third Party/XMLDictionary/XMLDictionary.m", language = "Objective-C"

```
这样就可以找到了……

3、imag lookup --type  查看类型可以打印类的属性和成员变量

```
(lldb) image lookup -t Model
Best match found in /Users/jiangliancheng/Library/Developer/Xcode/DerivedData/TLLDB-beqoowskwzbttrejseahdoaivpgq/Build/Products/Debug-iphonesimulator/TLLDB.app/TLLDB:
id = {0x30000002f}, name = "Model", byte-size = 32, decl = Modek.h:11, clang_type = "@interface Model : NSObject{
    NSString * _bb;
    NSString * _cc;
    NSString * _name;
}
@property ( getter = name,setter = setName:,readwrite,nonatomic ) NSString * name;
@end
"

```

4、其他不常用的方法，这里不再多说

### tips:

help & apropos
LLDB的命令其实还有很多，很多命令我也没玩过。就算玩过的命令，我们也非常容易忘记，下次可能就不记得是怎么用的了。还好LLDB给我们提供了2个查找命令的命令:help & apropos

参考文章：
[小笨狼与LLDB的故事](http://www.jianshu.com/p/e89af3e9a8d7)
