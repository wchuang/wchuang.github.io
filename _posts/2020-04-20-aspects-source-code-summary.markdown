---
layout: post
title: "Aspects 源碼梳理"
date: 2020-04-20T17:03:23+08:00
categories: [ios, aspects, runtime]
keywords: "iOS, aspects, runtime"
---

## 前言

之前曾在項目中使用 Aspects hook Objective-C 的一些特定方法以達到數據埋點並且上報的能力，當時對於它的具體實現及源碼細節並沒有深入追究，這次藉由重新複習 iOS runtime 的時候，重新梳理了下 Aspects 的代碼實現以及原理。

相信大家都知道 iOS 是門動態語言，它的 runtime 特性很具特色也是 iOS 的精華所在，但是對於如何使用這個神奇的特性，大家可能也是一知半解。Aspects 良好的向大家展示了如何運用 runtime 在實際的項目中，它提供的能力在 app 也是很實際會遇到的應用例子，所以想更加理解 runtime 的運用很推薦大家讀讀 Aspects 源碼。

## NSMethodSignature & NSInvocation

在 iOS 中調用方法除了

1. 一般方式：```[某個類的實例 方法名: 參數];```
2. PerformSelector 系列：```[某個類的實例 performSelector: @selector(方法:) withObject: 參數];```

還能利用 NSMethodSignature 以及 NSInvocation 完成方法調用。在 Aspects 中也用到了。

### NSMethodSignature 方法簽名

NSMethodSignature 如名稱所表示，它記錄了一個方法中的返回值以及參數，並且用了特定規則的符號字串來標示。有兩個方式可以創建 NSMethodSignature，

1. NSObject 中的 ```- (NSMethodSignature *)methodSignatureForSelector:(SEL)aSelector``` 或者 ```+ (NSMethodSignature *)instanceMethodSignatureForSelector:(SEL)aSelector```，利用 Selector 拿到方法簽名。
2. NSMethodSignature 也提供了方法可以利用特定的 char 組成字符串來表示方法的返回以及參數的類型，```+ (nullable NSMethodSignature *)signatureWithObjCTypes:(const char *)types;```

比如：NSString 的實例方法 ```- (BOOL)containsString:(NSString *)str;```
表示為：```c@:@```
1. c 表示返回值類型 Bool
2. @ 表示 receiver，也就是這個方法的對象（self）
3. : 表示 SEL (_cmd)
4. @ 表示第一個參數 NSString

其中 self 以及 _cmd 是固定隱含的，也就是說你只需要定義返回值以及參數類型。每個類型對應的字符官方已經定義好了，如果不知道可以用 ```@encode()```查詢，不需要 hard-code，詳細定義可以參考官方的對照表（https://developer.apple.com/library/archive/documentation/Cocoa/Conceptual/ObjCRuntimeGuide/Articles/ocrtTypeEncodings.html#//apple_ref/doc/uid/TP40008048-CH100）。

### NSInvocation 調用方式

蘋果官方表示
> An Objective-C message rendered as an object. 

也就是說蘋果把消息傳遞的消息體（包含了 target, selector, arguments 以及 return value）封裝成了一個 NSObject 方便使用。

你可以任意改變 NSInvocation 對象的 target（對象）, selector（方法標示），所以這意味著藉由修改 target 或者 selector 你可以把 invocation 對象重複的使用，而當 invocation 對象被 invoke 之後 return value 會自動被設置（因為已經調用了方法並且拿到回傳值了）。

初始化這個對象只能用 ```+ (NSInvocation *)invocationWithMethodSignature:(NSMethodSignature *)sig```，傳入 NSMethodSignature，不能用 NSObject 的 alloc+init。

還有就是 invocation 默認不會持有它的參數，如果初始化完 invocation 參數可能會消失的話，你需要自己調用 ```- (void)retainArguments```。

最後使用 ```- (void)invoke;``` 會根據 invocation 的參數（target, selector, methodSignature）就可以調用該方法並設置返回值了。

舉一個栗子：

```
- (void)run {
	NSMethodSignature *signature = [NSMethodSignature signatureWithObjCTypes:[@"v@:@" UTF8String]];
	NSInvocation *invocation = [NSInvocation invocationWithMethodSignature:signature];
	invocation.selector = NSSelectorFromString(@"test:");
	NSString *test = @"argu1";
	invocation.target = self;
	[invocation setArgument:&test atIndex:2]; // index從2開始第一個參數，0表示target，1是selector
	[invocation invoke];
}
    
- (void)test:(NSString *)test {
    NSLog(@"test: %@", test);
}
    
```

至於找不到方法的處理就不在這邊贅述了，來看下 Aspects 的源碼吧。

## Aspects 內部結構

### _AspectBlock


### AspectInfo

### AspectIdentifier

### AspectContainer

### AspectTracker

## Aspects hook 流程
