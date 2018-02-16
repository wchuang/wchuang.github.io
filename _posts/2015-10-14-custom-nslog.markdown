---
layout: post
title: "Custom NSLog"
date: 2015-10-14 01:21:14 +0800
comments: true
categories: "iOS"
keywords: "iOS, NSLog, obj-c, objective-c"
description: "custom NSLog-like macro"
---

	#ifdef DEBUG
	#   define DLog(fmt, ...) NSLog((@"%s [Line %d] " fmt), __PRETTY_FUNCTION__, __LINE__, ##__VA_ARGS__);
	#else
	#   define DLog(...)
	#endif
	
	#define ALog(fmt, ...) NSLog((@"%s [Line %d] " fmt), __PRETTY_FUNCTION__, __LINE__, ##__VA_ARGS__);
	
	#ifdef DEBUG
	#   define ULog(fmt, ...)  { UIAlertView *alert = [[UIAlertView alloc] initWithTitle:[NSString stringWithFormat:@"%s\n [Line %d] ", __PRETTY_FUNCTION__, __LINE__] message:[NSString stringWithFormat:fmt, ##__VA_ARGS__]  delegate:nil cancelButtonTitle:@"Ok" otherButtonTitles:nil]; [alert show]; }
	#else
	#   define ULog(...)
	#endif
	
加入以上的 code 到 .pch 檔。

這邊定義了三種不同的 log 方式

* DLog：相當於 NSLog，但它只有在 DEBUG 模式中才可以被使用。相較於 NSLog，自訂的 function 名稱及程式碼行數也會印出來。
* ALog：等同 NSLog，function 名稱及程式碼行數也會印出來。
* ULog：等於用 UIAlertView 印出 DLog。