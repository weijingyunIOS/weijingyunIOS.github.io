---
layout: post
title: "环信-语音通话--总结+反思"
date: 2015-12-14 19:55:46 +0800
comments: true
category:OpenSourceFrameWork

---

###简介

	这篇文章主要是记录了，我在基于环信的API 实现P2P语音的过程，包括中间犯的一些错误，总结了一部分经验。
	
###需求：

	我的App主要是针对美术生在学生和老师之间交流的一个工具，下面给出UI给的图，相信通过看图你就可以大概的理解到基本的流程
![p2p流程图](http://img.hoop8.com/attachments/1512/4603412352730.png)

	

###技术难点：
 
```
	做过IM的同学都知道，长连接是最重要的一部分，当然这里使用的是环信的API已经不需要考虑了，但是，通话过程中仍然存在着很多问题
	1、如何保证连接的存在？
		如果一方出现了网络的异常，如何保证另一方可以及时的挂断
	2、如何实现后台监听？
		环信对于消息的发送，已经支持了离线，但是对于语音和视频通话，如果用户点击Home键，应用进入后台，环信是不支持直接唤起的（iOS不可以，安卓可以）
	3、如何及时的更新用户的状态
		在我们的需求中会有一个在线的用户列表，用户在应用中有三种状态：在线、离线、忙碌、空闲，这几种状态要什么时候更新
	4、奇葩需求，无接听状态
		我们的需求后来改为，一方发起呼叫（未进行实质的呼叫） 另一方看到呼叫信息后，主动对发起呼叫方呼叫，发起方直接接听，不需要点击，接听按钮
```
 这些所谓的难点都是我认为，本人iOS能力一般，求轻喷……
 
 
###主要的功能点的是实现
 
 
####1、登录、登出

通常,我们都是在，自己的APP登录成功之后在进行环信的登录，APP登出成功之后会登出环信

```
	登录： asyncLoginWithUsername
	登出： asyncLogoffWithUnbindDeviceToken
```
	注意在这里我们可以设置,是否让换新自动登录，也可以理解为断线重连
在这里，由于我的语音通话要求用户在后台的时候也可以通话所以，这里设置自动登录

```
	自动登录：
	[[EaseMob sharedInstance].chatManager enableAutoLogin];
```
  如果你的应用不需要保持后台在线可以监听，系统进入后台（UIApplicationDidEnterBackgroundNotification）和进入前台（UIApplicationWillEnterForegroundNotification）的通知，在通知执行的事件中登录或者登出环信
  自动重连是否成功，你可以通过下面这个回调进行监听
  
```
  自动登录回调
  didAutoReconnectFinishedWithError
```
环信同时也给我们提供了一个方法，告诉我们当前用户环信是否登录

```
当前是否已有登录的用户
[[EaseMob sharedInstance].chatManager isLoggedIn];

是否连上聊天服务器
[[EaseMob sharedInstance].chatManager isConnected];

当前登录的用户信息
[[EaseMob sharedInstance].chatManager loginInfo];

```

总体来说，环信的登录，是存在一个步骤的：

	APP用户登录-->获取这个用户的环信信息（用户名/密码）-->登录环信
	APP用户登出 --> 环信登出
	

####2、语音拨打和接听

其实，在实际的项目中，我们还需要做很多的事，比如，要去维护一个在线用户的列表，那么就要求这个列表是可以实时刷新的，同时，为了增加用户的体验，我们应该有一个优先级，针对不同的用户，距离最近（根据IP地址），或者最近拨打过的，或者有主动邀请与他进行通话的这些因素考虑进去。
	在综合各种考量之后，可以拨打电话

```
EMCallSession *callSession = [[EaseMob sharedInstance].callManager asyncMakeVoiceCall:aMobID timeout:60.f error:&error];
```
注意：这个方法只是检查语音通话条件是否具备，不具备则返回错误信息（未进行实际的拨打动作）
在上面方法，返回的error为空时，切callSession不为空时才弹出通话界面

```
    if (!error && callSession) {
        //成功
        completion(nil);
        [[EaseMob sharedInstance].callManager
         							removeDelegate:self];
        //跳转
        ArtEMCallViewController *callVc = 		
        					[[ArtEMCallViewController alloc]
        					initWithSession:callSession 
  							isIncoming:NO];
        			
        callVc.easemobUser = aMobID;
        callVc.modalPresentationStyle = 	
        					UIModalPresentationFullScreen;
        
        ArtNavigationController *nav = 
  						[[ArtNavigationController alloc] 
  						initWithRootViewController:callVc];
        
        [ArtAppDelegate.window.rootViewController ]
        				presentViewController:nav
     					animated:YES completion:nil];
    }
    
```
在语音通话的开始阶段起，环信给我们提供了一个委托方法

```
实时通话过程中的 状态变化
- (void)callSessionStatusChanged:(EMCallSession *)callSession changeReason:(EMCallStatusChangedReason)reason error:(EMError *)error

```
方法解析：

(1)EMCallSession:通话实例

重要的属性：

```
sessionId:通话实例的id，唯一 可以理解为会话的id
sessionChatter： 通话对方的username
type：通话的类型（语音、视频、其他）
status：通话的状态
connectType：连接类型

```


(2)说到状态变化，就必须得说一下一共有哪几种状态：

```
 @brief 实时通话状态
 @constant eCallSessionStatusDisconnected 通话没开始
 @constant eCallSessionStatusRinging 通话响铃
 @constant eCallSessionStatusAnswering 通话双方正在协商
 @constant eCallSessionStatusPausing 通话暂停
 @constant eCallSessionStatusConnecting 通话已经准备好，等待接听
 @constant eCallSessionStatusConnected 通话已连接
 @constant eCallSessionStatusAccepted 通话双方同意协商

```
通过这些状态的变化，我们可以清楚的知道对方当前做了什么操作

(3)EMCallStatusChangedReason 状态变化的原因

```
/*!
 @enum
 @brief 实时通话结束原因
 @constant eCallReason_Null 正常挂断
 @constant eCallReason_Offline 对方不在线
 @constant eCallReason_NoResponse 对方没有响应
 @constant eCallReason_Hangup 对方挂断
 @constant eCallReason_Reject 对方拒接
 @constant eCallReason_Busy 对方占线
 @constant eCallReason_Failure 失败
 */
typedef NS_ENUM(NSInteger, EMCallStatusChangedReason){
    eCallReasonNull = 0,
    eCallReasonOffline,
    eCallReasonNoResponse,
    eCallReasonHangup,
    eCallReasonReject,
    eCallReasonBusy,
    eCallReasonFailure,
    eCallReason_Null = eCallReasonNull,
    eCallReason_Offline = eCallReasonOffline,
    eCallReason_NoResponse = eCallReasonNoResponse,
    eCallReason_Hangup = eCallReasonHangup,
    eCallReason_Reject = eCallReasonReject,
    eCallReason_Busy = eCallReasonBusy,
    eCallReason_Failure = eCallReasonFailure,
};

```
通过上面，监听状态的变化以及当前的状态，我们可以进行相应的动作
当callSession.status == eCallSessionStatusConnected时，我们在接听方弹起接听的界面

```
 if (callSession.status == eCallSessionStatusConnected) {
        //链接成功 这个状态是 有人打电话进来
        //判断 是否可以接电话
        [[EaseMob sharedInstance].callManager 
        							removeDelegate:self];
        ArtEMCallViewController *callController = 
        					[[ArtEMCallViewController alloc] 
        					initWithSession:callSession 
        					isIncoming:YES];
        					
       callController.easemobUser = 
       							callSession.sessionChatter;
       							
        callController.modalPresentationStyle = 
        				UIModalPresentationOverFullScreen;
        				
        UINavigationController *nav = 
        		[[UINavigationController alloc] 
        		initWithRootViewController:callController];
        		
        [ArtAppDelegate.window.rootViewController 
        			presentViewController:nav animated:NO 
        			completion:nil];
    }

```

当接收方弹出，通话界面的时候 我们可以采取以下几种操作：

1、拒接：

```
[[EaseMob sharedInstance].callManager 	
					asyncEndCall:aSessionID reason:reason];


```

2、接听

```
 [[EaseMob sharedInstance].callManager 
 							asyncAnswerCall:aSessionID];

```

3、挂断

```
[[EaseMob sharedInstance].callManager 
					asyncEndCall:aSessionID reason:reason];

```
静音或者免提的方法可以直接参考demo 这里不再赘述

到这里，通话就基本建立起来了；

`千万不要忘了添加委托`

```
	//语音通话的
    [[EaseMob sharedInstance].callManager
    				 addDelegate:self delegateQueue:nil];
    //其他类型消息
    [[EaseMob sharedInstance].chatManager 
    				addDelegate:self delegateQueue:nil];

```


####3、心跳包（文本消息）

(一)、消息的发送

不要怀疑，你没有看错，虽然我这里讲的是语音聊天，但是，在实时语音过程中，我为了保证连接，保证 一方因为异常原因 非正常的挂断，另一方可以及时感知到对方挂断，我在双方进行语音聊天的过程中，每隔一段时间会发送一次文本消息（类似心跳包），用来感知对方是否在线，以达到及时挂断的目的

首先应该知道环信目前支持的消息格式有：

	文字消息（EMChatText）、
	图片消息（EMChatImage）、
	位置消息（EMChatLocation）、
	语音消息（EMChatVoice）、
	视频消息（EMChatVideo）、
	文件消息（EMChatFile）、
	透传消息（EMChatCommand）
其实，总体看起来，所有的消息，环信都提供了一个对应的模型，然后 我们把这些模型转成响应的body 最后封装成一个统一的EMMessage对象 然后我们只需要调用相同的方法，就可以把这些消息发送出去了

```
    EMChatText *text = [[EMChatText alloc] initWithText:aText];
    EMTextMessageBody *body = [[EMTextMessageBody alloc] initWithChatObject:text];
    EMMessage *message = [[EMMessage alloc] initWithReceiver:aUserID bodies:@[body]];
    message.messageType = eMessageTypeChat;
    [[EaseMob sharedInstance].chatManager asyncSendMessage:message progress:nil];

```

好了，回到项目当中........
 因为我的文本消息只是当做心跳包，为了避免这些文本会显示在正常的聊天记录中，我写死了这个文本的内容

```
static NSString * const kHeartBeatMessage = @"--------------------******使用环信默认发送的文本消息，当做心跳作为对话时自己存活的标志！&&&&&&%%%%%%holo";

```
 之所以写的那么长，就是为了避免有相同的消息出现，方便我在文本消息显示的时候进行过滤

 注意：想要知道消息是否发送成功，环信也给我们提供了一个委托方法

```
-(void)didSendMessage:(EMMessage *)message error:(EMError *)error

```
在这里，我们可以对发送失败的消息进行处理

(二)、消息的接收
	消息的接受，环信也给我们提供了委托方法：

```
-(void)didReceiveMessage:(EMMessage *)message

```

这里接收到消息之后我们，需要根据消息的类型，解析消息的数据
	
	
```
// 收到消息的回调，带有附件类型的消息可以用SDK提供的下载附件方法下载
-(void)didReceiveMessage:(EMMessage *)message
{
    id<IEMMessageBody> msgBody = 
    					message.messageBodies.firstObject;
    					
    switch (msgBody.messageBodyType) {
        case eMessageBodyType_Text:
        {
            // 收到的文字消息
            NSString *txt = ((EMTextMessageBody 
            								*)msgBody).text;
            NSLog(@"收到的文字是 txt -- %@",txt);
        }
        break;
        case eMessageBodyType_Image:
        {
            // 得到一个图片消息body
            EMImageMessageBody *body =
            			 ((EMImageMessageBody *)msgBody);
            NSLog(@"大图remote路径 -- %@"   ,body.remotePath);
            NSLog(@"大图local路径 -- %@" ,body.localPath); 
            // 需要使用sdk提供的下载方法后才会存在
            NSLog(@"大图的secret -- %@"    ,body.secretKey);
            NSLog(@"大图的W -- %f ,大图的H -
            	%f",body.size.width,body.size.height);
            NSLog(@"大图的下载状态 -- 	
            	%lu",body.attachmentDownloadStatus);
 
 
            // 缩略图sdk会自动下载
            NSLog(@"小图remote路径 -- 
            				%@",body.thumbnailRemotePath);
            				
            NSLog(@"小图local路径 -- 
            				%@",body.thumbnailLocalPath);
            				
            NSLog(@"小图的secret --
            				%@",body.thumbnailSecretKey);
            
            NSLog(@"小图的W -- %f ,大图的H -- 
   							%f",body.thumbnailSize.width,
   							body.thumbnailSize.height);
   						
            NSLog(@"小图的下载状态 -- 
            			%lu",body.thumbnailDownloadStatus);
        }
        break;
        
        case eMessageBodyType_Location:
        {
            EMLocationMessageBody *body = 
            				(EMLocationMessageBody *)msgBody;
            				
            NSLog(@"纬度-- %f",body.latitude);
            
            NSLog(@"经度-- %f",body.longitude);
            
            NSLog(@"地址-- %@",body.address);
            
          }
            break;
         case eMessageBodyType_Voice:
            {
            // 音频sdk会自动下载
            EMVoiceMessageBody *body = 
            				(EMVoiceMessageBody *)msgBody;
            NSLog(@"音频remote路径 -- 
            				%@",body.remotePath);
            NSLog(@"音频local路径 -- 
            				\%@",body.localPath);
            // 需要使用sdk提供的下载方法后才会存在（音频会自动调用）
            NSLog(@"音频的secret -- %@",body.secretKey);
            NSLog(@"音频文件大小 -- %lld",body.fileLength);
            NSLog(@"音频文件的下载状态 -- 
            			%lu",body.attachmentDownloadStatus);
            NSLog(@"音频的时间长度 -- 
            			%lu",body.duration);
        }
        break;
        case eMessageBodyType_Video:
        {
            EMVideoMessageBody *body = 
            				(EMVideoMessageBody *)msgBody;
 
            NSLog(@"视频remote路径 -- %@",body.remotePath);
            
            NSLog(@"视频local路径 -- %@",body.localPath);
            
            // 需要使用sdk提供的下载方法后才会存在
            NSLog(@"视频的secret -- %@",body.secretKey);
            
            NSLog(@"视频文件大小 -- %lld",body.fileLength);
            
            NSLog(@"视频文件的下载状态 -- 
            			%lu",body.attachmentDownloadStatus);
            			
            NSLog(@"视频的时间长度 -- %lu",body.duration);
            
            NSLog(@"视频的W -- %f ,视频的H -- %f", 
            			body.size.width, body.size.height);
 
            // 缩略图sdk会自动下载
            NSLog(@"缩略图的remote路径 -- 
            				%@",body.thumbnailRemotePath);
            				
            NSLog(@"缩略图的local路径 -- 
            				%@",body.thumbnailRemotePath);
            				
            NSLog(@"缩略图的secret -- 
            				%@",body.thumbnailSecretKey);
            				
            NSLog(@"缩略图的下载状态 -- 
            			%lu",body.thumbnailDownloadStatus);
        }
        break;
        case eMessageBodyType_File:
        {
            EMFileMessageBody *body = 
            					(EMFileMessageBody *)msgBody;
            					
            NSLog(@"文件remote路径 -- %@",body.remotePath);
            
            NSLog(@"文件local路径 -- %@",body.localPath);
            
            // 需要使用sdk提供的下载方法后才会存在
            NSLog(@"文件的secret -- %@",body.secretKey);
            
            NSLog(@"文件文件大小 -- %lld",body.fileLength);
            
            NSLog(@"文件文件的下载状态 -- 
            			%lu",body.attachmentDownloadStatus);
        }
        break;
 
        default:
        break;
    }
}

```

由于 消息在我的P2P语音中，只做心跳作用，在接收到消息的时候，只是简单的判断一下，这条消息距离上一条消息的时间差，如果大于某一个界定的值（5s），表明已经5s没有收到对方的心跳，表明对方已经非正常挂断，自己主动挂断

这里我采用的是延时5s中，比较当前的消息和5s中之前的消息是否相同，这里的写法是采用[BlockKit](https://github.com/stevestreza/BlockKit)中的写法,这种写法的优点是`可以取消这个block的延时执行`

```
 self.exceptionHandleBlock = [self bk_performBlock:^(id obj) {
       if ([weakSelf.lastMessageID
       					 isEqualToString:message.messageId])
       		{
                  //5s钟未接受到新的消息
                  [weakSelf forceClose];
           }
                
} afterDelay:5.f];

```

到这里，我就可以基本上保证，通话双方，如果一方因为某些原因，异常挂断，另一方可以在5s中之后自动挂断


####4、图片的发送

经过上面的步骤，我们的通话基本可以建立起来且可以正常通话，那么我们接下来要实现的功能就是传图，这里传图的功能在和产品进行商量之后 我们也是采用的环信聊天传图（这样我们的服务器上可能就没办法保存通话过程中的图片）。

由上面的图可以看到，我们也是像其他的APP一样 可以拍照和从相册选取，这里就不在赘述，目光集中到环信上

需要注意的一点，`图片的发送可能会失败`，所以，我选择在发送方接收到图片发送成功的回调中才将图片显示（我们的APP中还有一个要求就是双方传图是有限制的），所以，我选择在图片发送成功的回调中添加图片可以避免这种情况。

图片的展示，我是使用了Collection做了一个简单的图片浏览器，这里也不再多说，这里有一个问题

[如何获取collectionview当前显示的item的indexpath](http://leewongsnail.github.io/blog/2015/12/05/ioskai-fa-zhong-de-xiao-ji-qiao/)
具体内容可以参考我的这篇博客里面总结了一些iOS开发中的小技巧，希望可以帮到你

经过上面的这几步，我们的功能基本上就可以实现了，但是还存在着很多潜在的问题
	


####5、后台呼起

1、主流语音聊天APP做法

```
	在搞这个功能之前呢，我首先是参考了QQ和微信这两个我们最经常使用的聊天APP，最新版的QQ和微信，都是采用了，在主呼叫方发起呼叫后，如果被叫APP处于后台的状态，那么，回想被叫发送推送，然后被叫点击推送消息之后，唤起APP，弹出通话界面
```

2、我的做法

```
	由于环信目前是不支持后台的，即使一方发起呼叫，另一方也无法感知（没有推送消息），所以走常规的路径是无法实现的。
	这里我采用的方法是，主叫方在发起语音呼叫的时候，先发送一条固定的消息（通心跳包类似），因为，APP进入后台之后消息是可以接受的，因此，我决定在接收到这条固定消息之后，发送一条本地的推送，然后让用户主动的去点击推送进而唤起APP。
	但是这样实现有一个问题，消息我是固定的（为了以后显示的时候可以控制），所以推送的内容，可能无法精确到个人

```

3、具体实现

```
(1)接受到消息之后发送一条本地的推送
if ([UIApplication sharedApplication].applicationState == 
    								UIApplicationStateBackground) {
        id<IEMMessageBody> msgBody = 
        					message.messageBodies.firstObject;
        					
        if (msgBody.messageBodyType == eMessageBodyType_Text) 
        {
        
            EMTextMessageBody *body = ((EMTextMessageBody 
            									*)msgBody);
            									
            if ([body.text isEqualToString:kVoicePushMessage]) 
            {
                //如果对方在后台 发送一条本地推送
                [self pushLocalNotification:message];
                return;
            }
        }
 }

(2)推送内容

- (void)pushLocalNotification:(EMMessage *)aMessage
{
    
    //发送本地推送
    UILocalNotification *notification = 
    						[[UILocalNotification alloc] init];
    						
    notification.fireDate = [NSDate date]; //触发通知的时间
    notification.alertBody = 				
   		NSLocalizedString(@"receiveMessage", @"老师正在呼叫....");
   		
    notification.soundName = @"callRing.mp3";
    
    notification.userInfo = @{@"type":@(10001)};
    
    notification.hasAction = NO;
    notification.timeZone = [NSTimeZone defaultTimeZone];
    
    //发送通知
    [[UIApplication sharedApplication]
    				 scheduleLocalNotification:notification];
    				 					
    UIApplication *application = [UIApplication 
    										sharedApplication];
    										
    application.applicationIconBadgeNumber += 1;

}

(3)收到推送后弹出控制器

- (void)application:(UIApplication *)application didReceiveLocalNotification:(UILocalNotification *)notification
{
    ArtP2PStuCallingController *vc = 
    				[[ArtP2PStuCallingController alloc] init];
    				
    ArtNavigationController *nav = [[ArtNavigationController 						alloc] initWithRootViewController:vc];
    
    UIViewController *curVC = [self 
    							getCurrentRootViewController];
    
    dispatch_after(dispatch_time(DISPATCH_TIME_NOW, (int64_t)(0.1f * NSEC_PER_SEC)),
      dispatch_get_main_queue(), ^{
        [curVC.navigationController presentViewController:nav
        							 animated:NO completion:nil];
      });
}

```
这样我们就可以基本上实现了，类似微信或者QQ的后台功能

####6、疑难杂症

1、出现第一次可以接通无反应的情况

```
原因1：需要增加对于通知的监听
 [[NSNotificationCenter defaultCenter] 
 						addObserver:self 
 						selector:@selector(callOutWithChatter:) 
 						name:@"callOutWithChatter" object:nil];
 						
 [[NSNotificationCenter defaultCenter] 
   						addObserver:self 
   						selector:@selector(callControllerClose:) 
   						name:@"callControllerClose" object:nil];
具体的实现方法可以参考环信的Demo

```
2、上次通话未正常结束，下一次通话无反应

```

	环信要求，如果本次通话为异常挂断，那么需要我们在程序中，主动调用一下挂断的方法，如果上次通话未能挂断，那么将无法接通新的通话

	[[EaseMob sharedInstance].callManager 
						asyncEndCall:aSessionID reason:reason];


```

3、对于我们的需求，要求通话建立后被叫方直接接听

这里，我的方法是在创建控制器的方法
	
```
- (instancetype)initWithSession:(EMCallSession *)session
                     isIncoming:(BOOL)isIncoming;

```
根据是主叫还是被叫，在视图显示完成之后，调用一下接听的方法

4、用户状态的更新

正常情况：

	登录成功(在线空闲)-- > 呼叫对方（在线忙碌）--> 呼叫挂断（在线空闲）-->环信退出（离线）
	
异常情况：

	登录成功(在线空闲) --> 呼叫对方(在线忙碌) - >呼叫挂断异常（网络问题）--> 环信重连-->重连登录成功(在线空闲)
	

5、注意：

	经测试，环信demo的callviewcontroller 没有dealloc，而且 如果没有dealloc,会导致很多的资源没办法释放，所以这里特别注意这个控制器的释放
	
