---
layout: post
title: "iOS运行时常用方法"
date: 2014-11-16 13:51:49 +0800
comments: true
categories: 
---
###一、Runtime常用使用情景
####1、KVC 属性不存在时崩溃的解决
	RuntimeObj *obj = [[RuntimeObj alloc]init]; 
	[obj setValue:@"value4Name" forKey:@"objName"];
	//	RuntimeObj 没有objName这个属性
	这种情况下会直接崩溃，对于这个情况可以使用运行时来解决
	解决方法：检查某个类是否含有某个属性
	-(BOOL)hasAttribute:(NSString *)attName
	{
   	 	BOOL flag = NO;
    	u_int count;
    	Ivar *ivars = class_copyIvarList([self class], &count);
    
    	for (int i = 0; i < count ; i++)
    	{
        	const char* propertyName = 												ivar_getName(ivars[i]);
        	NSString *strName = [NSString  									stringWithCString:propertyName 							encoding:NSUTF8StringEncoding];
        	if ([attName isEqualToString:strName]) {
            	flag = YES;
        	}
       	 NSLog(@"===%@",strName);
    	}
    	return flag;
	}
	方法解析：
	Ivar 原型是typedef struct objc_ivar *Ivar; 
	Ivar *ivars = class_copyIvarList([self class], 	&amp;count); 
	返回的是某个类所有属性或变量原型
	 ivar_getName 返回的是没有 Ivar 结构体的名字，即变量的名字 原型 const char *ivar_getName(Ivar v)；
	 //another way to get property list
	- (void)getPropertyList
	{
    	u_int count;
    	objc_property_t* properties= 						class_copyPropertyList([self class], &count);
    	for (int i = 0; i < count ; i++)
    	{
        		const char* propertyName = 								property_getName(properties[i]);
        		NSString *strName = [NSString  					stringWithCString:propertyName 					encoding:NSUTF8StringEncoding];
        		NSLog(@"===%@",strName);
    	}
	}

####2、动态创建函数
	
	void dynamicMethod(id self, SEL _cmd)
	{
		//后调用
    	printf("SEL %s did not exist\n",sel_getName(_cmd));
	}

	+ (BOOL) resolveInstanceMethod:(SEL)aSEL
	{
    	//先调用 判断有没有这个方法 如果没有就创建
    	class_addMethod([self class], aSEL, 						(IMP)dynamicMethod, "v@:");
    	return YES;
	}
	解析：
	* + (BOOL) resolveInstanceMethod:(SEL)aSEL 是在调用此类方法时，如果没有这个方法就会掉这个函数。
	* class_addMethod 就是动态给类添加方法 原型 BOOL class_addMethod(Class cls, SEL name, IMP imp, const char *types) 
	调用方：
	 DynamicMethod *method = [[DynamicMethod alloc] init];
    [method performSelector:@selector(dynamicMethod:)];

#### 替换已有函数
	void demoReplaceMethod(id SELF, SEL _cmd)
	{
    	NSLog(@"demoReplaceMethod");
	}

	-(void)replaceMethod
	{   
		 Class strcls = [self class];
    	SEL  targetSelector = @selector(targetRelplacMethod);
    	class_replaceMethod(strcls,targetSelector,	(IMP)demoReplaceMethod,NULL);
	}

	+ (void)replace
	{
    	Method m1 = class_getInstanceMethod(self, 		@selector(targetRelplacMethod));
    	Method m2 = class_getInstanceMethod(self, 		@selector(replaceMethod));
    	method_exchangeImplementations(m1, m2);
	}

	-(void)targetRelplacMethod
	{
   	 NSLog(@"targetRelplacMethod");
	}
  	解析：
		1. class_replaceMethod 方法就是动态替换Method的函数，原型 	IMP 
		2. class_replaceMethod(Class cls, SEL name,IMP imp, 	const char *types) 返回值就是一个新函数的地址（IMP指针）
		Method 结构体
		struct objc_method {
    		SEL method_name     OBJC2_UNAVAILABLE; //方法名
    		char *method_types  OBJC2_UNAVAILABLE; //方法？？
    		IMP method_imp      OBJC2_UNAVAILABLE; //方法地址
		}
####动态挂载对象
	//挂载对象所需要的参数（UIAlertView挂载对象） 
	static const char kRepresentedObject; 
	-(void)showAlert:(id)sender 
	{ 
    	UIAlertView *alert = [[UIAlertView 				alloc]initWithTitle:@"提示" message:message 			delegate:self cancelButtonTitle:@"取消" 			otherButtonTitles:@"去看看", nil]; 
    	alert.tag = ALERT_GOTO_TAG; 
    	objc_setAssociatedObject(alert, 				&amp;kRepresentedObject, 
    	@"我是被挂载的", 
    	OBJC_ASSOCIATION_RETAIN_NONATOMIC); 
    	[alert show]; 
	} 
	
	-(void)alertView:(UIAlertView *)alertView 					clickedButtonAtIndex:(NSInteger)buttonIndex 
	{ 
    		if (buttonIndex == 1) { 
        		NSString *str = 								objc_getAssociatedObject(alertView, 
                                 kRepresentedObject); 
        	NSLog(@"%@",str) 
    		}  
	}
	这个方法主要用于类似给一个类增加一个属性，这个属性有一个标示符，在其他地方用到的时候可以通过这个标示符去去这个值
	解析：
	1. static const char kRepresentedObject; 这个只是一个标	记，但是必不可少 具体什么作用没做过调研，我觉得应该就是你挂载的一个	标记 Runtime 应该会根据这个标记来区别被挂载对象是挂载在哪个实例	上。
	2. objc_setAssociatedObject 动态设置关联对象（也就是挂载）。
	3. objc_getAssociatedObject 动态获取关联对象 看到没有这里也要	传 kRepresentedObject 这个标记，好像有点证明我前面的猜想了

###二、API简析
####1、方法的替换
 	 Replaces the implementation of a method for a given class.
	 * 
 	* @param cls The class you want to modify.
 	* @param name A selector that identifies the method whose 	implementation you want to replace.
 	* @param imp The new implementation for the method 	identified by name for the class identified by cls.
 	* @param types An array of characters that describe the 	types of the arguments to the method. 
 	*  Since the function must take at least two arguments—	self and _cmd, the second and third characters
 	*  must be “@:” (the first character is the return type).
 	* 
 	* @return The previous implementation of the method 	identified by \e name for the class identified by \e cls.
 	* 
 	* @note This function behaves in two different ways:
 	*  - If the method identified by \e name does not yet 	exist, it is added as if \c class_addMethod were called. 
 	*    The type encoding specified by \e types is used as 	given.
 	*  - If the method identified by \e name does exist, its 	\c IMP is replaced as if \c method_setImplementation were 	called.
 	*    The type encoding specified by \e types is ignored.
 	*/
	OBJC_EXPORT IMP class_replaceMethod(Class cls, SEL name, IMP imp, 
                                    const char *types) 
     __OSX_AVAILABLE_STARTING(__MAC_10_5, __IPHONE_2_0);
     
 解析：
 OBJC_EXPORT IMP class_replaceMethod(Class cls, SEL name, IMP imp,const char *types)

	 参数1、你要修改的类（要替换方法的类）
	 参数2、你要替换的方法名
	 参数3、想要替换原有类的方法的新方法
	 参数4、方法参数类型描述的数组
####2、获取属性列表 
 	* Describes the instance variables declared by a class.
 	* 
 	* @param cls The class to inspect.
 	* @param outCount On return, contains the length of the 	returned array. 
 	*  If outCount is NULL, the length is not returned.
 	* 
 	* @return An array of pointers of type Ivar describing 	the instance variables declared by the class. 
 	*  Any instance variables declared by superclasses are 	not included. The array contains *outCount 
 	*  pointers followed by a NULL terminator. You must free 	the array with free().
 	* 
 	*  If the class declares no instance variables, or cls 	is Nil, NULL is returned and *outCount is 0.
	OBJC_EXPORT Ivar *class_copyIvarList(Class cls, unsigned 	int *outCount) 
     __OSX_AVAILABLE_STARTING(__MAC_10_5, __IPHONE_2_0);
     解析
     OBJC_EXPORT Ivar *class_copyIvarList(Class cls, unsigned 	int *outCount)
     
     	参数1：你要获取哪个类的属性列表
     	参数2：返回这个属性列表中属性的个数
####3、获取属性列表的另一种方法
	 * Describes the properties declared by a class.
	 * 
 	* @param cls The class you want to inspect.
 	* @param outCount On return, contains the length of the 	returned array. 
 	*  If \e outCount is \c NULL, the length is not 	returned.        
 	* 
 	* @return An array of pointers of type \c 		objc_property_t describing the properties 
 	*  declared by the class. Any properties declared by 	superclasses are not included. 
 	*  The array contains \c *outCount pointers followed by 	a \c NULL terminator. You must free the array with \c 	free().
 	* 
 	*  If \e cls declares no properties, or \e cls is \c 	Nil, returns \c NULL and \c *outCount is \c 0.
	OBJC_EXPORT objc_property_t 					*class_copyPropertyList(Class cls, unsigned int 										*outCount)
     __OSX_AVAILABLE_STARTING(__MAC_10_5, __IPHONE_2_0);
   	解析：
   		OBJC_EXPORT objc_property_t 					*class_copyPropertyList(Class cls, unsigned int 										*outCount)
   		
   		参数1：要获取哪个类的属性列表
   		参数2、返回这个类的属性的个数
   	区别：
   	copyPropertyList 只能获取@property声明的属性
   	copyIvarList 可以获取所有的包括使用{}声明的属性

####动态增加一个方法
 	* Adds a new method to a class with a given name and 	implementation.
 	* 
 	* @param cls The class to which to add a method.
 	* @param name A selector that specifies the name of the 	method being added.
 	* @param imp A function which is the implementation of 	the new method. The function must take at least two 	arguments—self and _cmd.
 	* @param types An array of characters that describe the 	types of the arguments to the method. 
 	* 
 	* @return YES if the method was added successfully, 	otherwise NO 
 	*  (for example, the class already contains a method 	implementation with that name).
 	*
 	* @note class_addMethod will add an override of a 	superclass's implementation, 
 	*  but will not replace an existing implementation in 	this class. 
 	*  To change an existing implementation, use 	method_setImplementation.
 	*/
	OBJC_EXPORT BOOL class_addMethod(Class cls, SEL name, 	IMP imp, const char *types) 
     __OSX_AVAILABLE_STARTING(__MAC_10_5, __IPHONE_2_0);
    解析：
    	OBJC_EXPORT BOOL class_addMethod(Class cls, SEL name, 	IMP imp, const char *types) 
    	参数1：想要添加方法的类
    	参数2：要添加的方法的方法名
    	参数3：动态生成的方法（必须包含至少两个参数self和cmd）
    	dynamicMethod(id self, SEL _cmd) 
    	参数4：返回的数组，描述方法的参数
    添加成功返回YES 否则返回NO
    	注意：class_addMethod会覆盖父类的方法实现，但是不会覆盖当前类中已经存在的方法，如果想修改当前类中存在的方法那么使用method_setImplementation
####设置关联对象
 	* Sets an associated value for a given object using a 	given key and association policy.
 	* 
 	* @param object The source object for the association.
 	* @param key The key for the association.
 	* @param value The value to associate with the key key 	for object. Pass nil to clear an existing association.
 	* @param policy The policy for the association. For 	possible values, see “Associative Object Behaviors.”
 	* 
 	* @see objc_setAssociatedObject
 	* @see objc_removeAssociatedObjects
	OBJC_EXPORT void objc_setAssociatedObject(id object, 	const void *key, id value, objc_AssociationPolicy 			policy)
    __OSX_AVAILABLE_STARTING(__MAC_10_6, __IPHONE_3_1);
    解析：
    OBJC_EXPORT void objc_setAssociatedObject(id object, 	const void *key, id value, objc_AssociationPolicy 			policy)
    给一个已经存在的对象设置一个关联对象，可以看做这个对象的一个属性，关联的时候必须设置一个key
    参数1：要关联的对象
    参数2：要关联的对象的key
    参数3：要关联的对象key的值，传nil可以清空这个对象的值
    参数4: 关联的策略
    typedef OBJC_ENUM(uintptr_t, objc_AssociationPolicy) {
   	 OBJC_ASSOCIATION_ASSIGN = 0,           /**< Specifies a 	weak reference to the associated object. */
    OBJC_ASSOCIATION_RETAIN_NONATOMIC = 1, /**< Specifies a 	strong reference to the associated object. 
                                            *   The 	association is not made atomically. */
    OBJC_ASSOCIATION_COPY_NONATOMIC = 3,   /**< Specifies 	that the associated object is copied. 
                                            *   The 	association is not made atomically. */
    OBJC_ASSOCIATION_RETAIN = 01401,       /**< Specifies a 	strong reference to the associated object.
                                            *   The 	association is made atomically. */
    OBJC_ASSOCIATION_COPY = 01403          /**< Specifies 	that the associated object is copied.
                                            *   The 	association is made atomically. */
	};
####获取关联对象的值 
 	* Returns the value associated with a given object for a 	given key.
 	* 
 	* @param object The source object for the association.
 	* @param key The key for the association.
 	* 
 	* @return The value associated with the key \e key for 	\e object.
 	* 
 	* @see objc_setAssociatedObject
	OBJC_EXPORT id objc_getAssociatedObject(id object, const 	void *key)
    __OSX_AVAILABLE_STARTING(__MAC_10_6, __IPHONE_3_1);
    解析：
    	OBJC_EXPORT id objc_getAssociatedObject(id object, 	const 	void *key)
    	获取关联对象的值
    参数1：要获取的关联对象
    参数2：要获取的管理属性的key
    使用场景可以是，给一个对象绑定一个属性并设置一个key值，然后在另一个地方，通过key值取出设置的属性
