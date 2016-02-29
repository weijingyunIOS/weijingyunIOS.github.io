---
layout: post
title: "iOS 跳转到设置中的某个界面的方法"
date: 2015-12-05 21:25:26 +0800
comments: true
category: Objective-C

---
前言：
	在APP开发中 我们会经常遇到，要跳转到设置的某个界面提示用户去设置（开启定位或者其他功能），下面详细的介绍了，跳转到每一个界面的方法
	
#### 1、方法
例子：跳转到定位服务

```
//定位服务设置界面
NSURL *url = [NSURL URLWithString:@"prefs:root=LOCATION_SERVICES"];
if ([[UIApplication sharedApplication] canOpenURL:url])
{
    [[UIApplication sharedApplication] openURL:url];

```
其他界面也是一个类型，不过URL改变一下就可以

#### 2、跳转类型

```
About — prefs:root=General&path=About	//关于本机
Airplane Mode On — prefs:root=AIRPLANE_MODE  //飞行模式
Auto-Lock — prefs:root=General&path=AUTOLOCK //屏幕锁定
Bluetooth — prefs:root=General&path=Bluetooth	//蓝牙设置
Date & Time — prefs:root=General&path=DATE_AND_TIME //日期时间
FaceTime — prefs:root=FACETIME	//FaceTime 设置
General — prefs:root=General	//通用
Keyboard — prefs:root=General&path=Keyboard	//键盘
iCloud — prefs:root=CASTLE	 //iCloud 用户设置
iCloud Storage & Backup  
prefs:root=CASTLE&path=STORAGE_AND_BACKUP //iCloud 存储空间
International — prefs:root=General&path=INTERNATIONAL	//语言地区
Location Services — prefs:root=LOCATION_SERVICES //定位服务
Network — prefs:root=General&path=Network	//通用并非网络
Notes — prefs:root=NOTES	//备忘录
Notification — prefs:root=NOTIFICATIONS_ID //通知	
Photos — prefs:root=Photos //照片与相机
Profile — prefs:root=General&path=ManagedConfigurationList
// 描述文件
Reset — prefs:root=General&path=Reset //还原
Sounds — prefs:root=Sounds	//声音
Software Update — 	prefs:root=General&path=SOFTWARE_UPDATE_LINK //软件更新
Wallpaper — prefs:root=Wallpaper //墙纸
Wi-Fi — prefs:root=WIFI	//WiFi

```

总结：

```
	希望这篇文章可以帮到你！
```

参考文章:[iOS开发之如何跳到系统设置里的各种设置界面](http://www.superqq.com/blog/2015/12/01/jump-setting-per-page/)
