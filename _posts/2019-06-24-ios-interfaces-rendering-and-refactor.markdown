---
layout: post
title: "iOS 介面渲染及優化"
date: 2019-06-24T23:56:57+08:00
comments: true
categories: "iOS, ui rendering, off-screen, on-screen, framebuffer"
keywords: "iOS, ui rendering, off-screen, on-screen, framebuffer"
---

## 前言

這篇文章主要想要整理一下 iOS 上介面如何渲染以及怎麼優化，雖說是 iOS，但其實原理都是一樣的。分為下列三個知識點來討論：

1. 圖片如何從 raw data 呈現在螢幕上？
2. 哪些情況下會造成介面顯示不流暢？
3. 要如何解決這些問題？

## 圖片如何顯示?

圖片如何從最一開始拿到的 raw data，最後展現在螢幕上讓使用者看到？所謂的 raw data 可能會從網路或者 bundle 的一張圖片取得，然後經由 CPU 計算解碼，GPU 渲染至緩存區，最後與硬體同步顯示在螢幕上。概觀圖如下：

![V-sync](https://blog.wchuang.cc/images/vsync/vsync.001.jpeg)

傳統 CRT 顯示器在顯示影像會由上到下垂直掃描，掃描完成後就會呈現一個 Frame 的完整畫面，同時電子槍會回到初始位置繼續下一次的掃描。為了讓系統 Controller 知道什麼時候掃描完成並要開始下一次掃描，顯示器會發出定時信號。

定時信號又分為

```
1. 水平同步信號(H-Sync): 當顯示器完成當前的行掃描並要換到新的一行時。
2. 垂直同步信號(V-Sync): 當一個 Frame 完成掃描並繪製完成，電子槍回復到初始位置，準備下一個 Frame 的掃描時。
```

垂直同步信號(V-Sync)也就是所謂的 FPS，顯示器刷新的頻率，也是我們用來評估是否畫面造成卡頓的因素之一。

當系統 Controller 收到來自顯示器的 V-Sync 訊號量後，它就知道螢幕已經完成一次顯示。這時候它會向 FrameBuffer 讀取下一個 Frame 的資料。

Framebuffer 在 Wiki 的定義：

```
A framebuffer (frame buffer, or sometimes framestore) is a portion of RAM[1]
 containing a bitmap that drives a video display. 
It is a memory buffer containing a complete frame of data.[2] 

Modern video cards contain framebuffer circuitry in their cores. 
This circuitry converts an in-memory bitmap into a video signal 
that can be displayed on a computer monitor. 
```
所以我們知道它是一塊記憶體緩存區，負責存放由 CPU/GPU 處理後的一塊完整 Frame data。

而現代的 framebuffer 幾乎都為雙緩存機制。這個機制下，GPU 會先渲染一個 Frame 的資料存入第一個 buffer 裡，讓 Controller 可以讀取。當下一個 Frame 渲染完成後，GPU 會把 Controller 的指針指向第二個 buffer，讓 Controler 讀取第二個 buffer 的資料。

但是如果 GPU 每次渲染完第二個 Frame 後就讓 Controller 讀取第二個 buffer 的資料，有可能因為 Controller 還未讀取完第一個 Frame 的資料就切換到第二個，造成了畫面畫面撕裂現象。畫面的上半部顯示舊的影像，下半部顯示了新的影像。

![Screen_tearing](https://blog.wchuang.cc/images/vsync/screen_tearing.jpg)

那麼要如何解決這個問題？

GPU 也有 V-Sync 的同步機制，當 V-Sync 開啟時，GPU 會等待顯示器的 V-Sync 信號發出後才進行下一個 Frame 的渲染以及緩存區更新。這樣便解決了畫面撕裂問題，但是也因為沒辦法先行處理下一個 Frame 的渲染，所以可能會有延遲的情況產生。

## 為什麼卡頓?

在 iOS 中，當一個 V-Sync 通知到來的時候，Core Animation 會利用 CADisplayLink 通知 APP，APP main thread 會開始在 CPU 計算顯示內容，如：view 的創建、Frame 的計算、圖片解碼、文本繪製等。然後 CPU 會將計算好的內容提交給 GPU，由 GPU 進行變換、合成、渲染，最後將渲染的結果存到緩存區中，讓 Controller 讀取、顯示在螢幕上並等待下一次 V-Sync 到來。

上述是一個完整的 V-Sync 圖像處理流程。但是如果在一次的 V-Sync 時間內，CPU 及 GPU 沒辦法即時完成任務並提交到緩存區內，這一次的 Frame 就會被丟棄。也就是說在下一個 V-Sync 來臨時，Controller 依舊讀取到 buffer 內的舊資料並顯示，這就是造成介面卡頓的原因。

所以，CPU 或 GPU 不論哪個阻礙了顯示流程，都會造成掉幀情況。所以開發時，需要分別對 CPU 及 GPU 壓力進行評估及優化來避免這類的問題。

## 如何解決?

要知道如何解決前，需要先了解 CPU 及 GPU 在處理圖像時，都做了哪些事情。

### CPU

1. 對象創建

	對象的創建會涉及到內存的分配、屬性調整、計算 retain count 等的操作，相對來說較耗用 CPU 資源。所以可以盡量利用比較**輕量的對象來取代**。如：用 struct 取代 class 來定義 model，或者當不需要響應使用者觸摸事件的控件，可以使用 CALayer 來代替 UIView。**多復用已創建的對象**，可以建立緩存存放。


2. 對象調整

	對象的屬性調整也是很消耗 CPU 的地方。由於 UIView 關於顯示相關的屬性，如 frame/bounds/transform 等實際上也是映射到 CALayer 屬性，所以修改 UIView 的這些屬性的時候，實際上消耗的性能遠大於其他的屬性。而當 UIView 進行層級調整的時候，UIView 及 CALayer 之間也會出現多次方法調用以及通知。

	我這邊對 CALayer 做了擴展並加了自定義屬性來測試，發現 **CALayer 會在 runtime 時自己添加上屬性方法**。若是對 UIView 做擴展及自定義屬性的話，在調用時會出現找不到方法造成 crash，你需要自己在 runtime 添加屬性方法。

	```
	// MyLayer
	@interface MyLayer : CALayer
	@end
	
	@interface MyLayer (MyTest)
	@property (copy, nonatomic) NSString *name;
	@end
		
	@implementation MyLayer
	@end
		
	@implementation MyLayer (MyTest)
	@end
	
	// Test
	MyLayer *layer = [[MyLayer alloc] init];
	[self.view.layer addSublayer:layer];
	layer.backgroundColor = UIColor.redColor.CGColor;
	    
	for (int i = 0; i < 1000000; i++) {
	    layer.name = [NSString stringWithFormat:@"MyLayer: %d", i];
	    layer.frame = CGRectMake(50, 50, 10, 10);
	    NSLog(@"layer name: %@", layer.name);
	}
	```

	```
	// MyView
	@interface MyView : UIView
	@end
	
	@interface MyView (MyTest)
	@property (copy, nonatomic) NSString *viewName;
	@end
		
	@implementation MyView
	@end
		
	@implementation MyView (MyTest)
	@end
	
	// Test
	MyView *view = [[MyView alloc] init];
	view.frame = CGRectMake(0, 0, 100, 100);
	view.viewName = @"MyView"; // Crash!!
	[self.view addSubview:view];
	```

	觀察調用棧會發現在設定自定義屬性 `name` 的時候，底層做了很多事情，包含了添加屬性方法、尋找屬性方法、設定屬性內容、發送 KVO 通知、`CA::Layer::begin_change()`、`CA::Layer::end_change()`、`CA::Transaction::ensure_compact()`、加鎖等的操作。

	![Custom_layer](https://blog.wchuang.cc/images/vsync/custom_layer.png)

3. 對象銷毀

	對象銷毀雖然消耗的資源不明顯，但若是累積起來也是不容忽視。尤其是一次釋放大量對象的時候。也可以盡可能的放在 background thread 執行。
	
4. 佈局計算

	佈局計算是最常見消耗系統資源的地方。如果能先在 background thread 計算好並作緩存就能避免這些問題。但上述也提到，調整佈局相關屬性(frame/bounds/center)其實都會對應到 CALayer 層級操作，還有同步通知問題，是非常消耗性能的地方。所以最好**先做預先計算及緩存，並且一次性的調整，而不要多次、頻繁的改變這些屬性**。

5. 文本計算

	如果在介面上有大量的文本需求（如：facebook news feed 或微信朋友圈），常常需要計算文本寬、高度，可以利用 `NSAttributedString` 中的 `[string boundingRectWithSize:options:context:];`、`[string drawWithRect:options:context:];` 這兩個方法來實現以提升效能，由於是 thread safe，所以可以放在 background thread 執行。

6. 文本渲染

	螢幕上能看到的所有文本內容控件，包含 UIWebView，其底層都是通過 `CoreText` 來排版並繪製成 `Bitmap` 顯示。常見的控件（UILabel、UITextView）等，其排版以及繪製都是在主線程進行。所以當有大量的文本需要顯示的時候，CPU 的壓力就會非常大。
	
	對此解決方式可以__自定義一個文本控件，並用`TextKit`或`CoreText`對文本異步繪製__。`CoreText` 對象創建好後，能直接取得文本的寬度高度等訊息。**避免了多次計算（調整 UILabel 大小的時候計算一次、UILabel 繪製的時候內部又再算一次）**。且`CoreText`對象佔用內存較少，可以緩存下來做多次復用。

7. 圖片解碼

	當你用 UIImage 或 CGImageSource 的那幾個方法創建圖片的時候，圖片資料並不會馬上被解碼。當圖片設置到 UIImageView 或 CALayer.contents 時，並且 CALayer 被提交到 GPU 渲染前，CGImage 中的資料才得以解碼。由於這一步是 UI 操作，會發生在主線程，且不可避免。優化方式可以先在 **background thread 把圖片繪製到 CGBitmapContext 中，然後由 Bitmap 直接創建圖片**。

8. 圖像繪製

	圖像繪製指的是使用以 CG 開頭的底層方法，把圖像繪製到畫布中，然後從畫布創建圖片並顯示的一個過程。最常見的地方就是在`[UIView drawRect:]`方法裡面。由於 CoreGraphic 的方法是 thread safe 的，所以可以在 background thread 繪製。
	
	```
	dispatch_async(dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0), ^{
	    CGContextRef contextRef = CGBitmapContextCreate();
	    // Draw your context.
	    CGImageRef imageRef = CGBitmapContextCreateImage(contextRef);
	    CGContextRelease(contextRef);
	    dispatch_async(dispatch_get_main_queue(), ^{
	        layer.contents = (__bridge id _Nullable)(imageRef);
	        CGImageRelease(imageRef);
	    });
	});
	```

### GPU

1. 紋理渲染

	所有的 Bitmap，不論是圖片或是文本的內容，最終都是要由內存提交到顯示卡記憶體，並綁定為 GPU 的 Texture。所以不管是提交的過程或是 GPU 調整和渲染 Texture 的過程，都要消耗不少 GPU 的資源。
	
	當在較短時間內顯示大量的圖片時（如 UITableView 存在很多圖片並快速滑動的時候），CPU 佔用率很低，GPU 佔用率非常高，此時介面依然會掉幀。避免此情況的方法只能**避免在短時間內的大量圖片顯示，盡可能將多張圖片合併成一張顯示**。
	
	當圖片過大，超過 GPU 的最大 Texture size 時，圖片需要先由 CPU 進行預處理，這對 CPU 及 GPU 都會帶來額外的資源消耗。可以從這個網站查看各個 iPhone 機型對應的 [MAX Texture size](http://iosres.com/)。

2. 視圖混合

	當多個 View（CALayer）重疊在一起的時候，GPU 會先把它們混合到一起。如果視圖結構過於複雜，混合的過程也會消耗很多 GPU 資源。為了減少這種情況的資源消耗，可以盡量**減少 view 的層級以及數量，並設置 opaque 為不透明，減少計算 background color 時的 alpha 合成**。也可以將多張視圖預先渲染為一張圖片顯示。

3. 圖形生成

	CALayer 的 border、圓角、陰影、遮罩、CASharpLayer 的矢量圖形顯示，通常會造成離屏渲染（Off-Screen Rendering)。
	
	GPU 的屏幕渲染分為兩種：

	1. On-Screen Rendering：指的是 GPU 的渲染操作在當前用於顯示的緩存區中。
	2. Off-Screen Rendering：指的是 GPU 的渲染操作不在當前用於顯示的緩存區中，而是另外建立一個緩存區做渲染操作。如果我們**重寫了 drawRect: 方法，並使用了 Core Graphics 方法進行繪製得到 Bitmap 後在交給 GPU 顯示**，也算是一種離屏渲染，因為是由 CPU 渲染。

	

	而離屏渲染（Off-Screen Rendering）的代價是很高的，原因在於：

	1. **創建新的緩存區**：系統必須多花費資源重新創建。
	2. **上下文切換**：必須先由 On-Screen 切換到 Off-Screen 環境，待 Off-Screen Rendering 結束後，把渲染結果同步到 On-Screen 環境。


	我們要避免離屏渲染造成系統無謂的資源浪費，可以設置 `CALayer.shouldRasterize` 屬性為 YES，使得在離屏渲染發生的時候會將渲染後的內容緩存起來，在下一個 Frame 渲染時可以直接復用。但如果你又同時設置了 border、圓角 等其他屬性，緩存將不會起作用。
	
	所以最好的方式，還是要避免上述的圓角、border、mask 等屬性的調用，可以用 CPU 渲染來代替。

## 參考資料

1. [Framebuffer](https://en.wikipedia.org/wiki/Framebuffer)
2. [Screen_tearing](https://en.wikipedia.org/wiki/Screen_tearing)
3. [CADisplayLink](https://developer.apple.com/documentation/quartzcore/cadisplaylink)
4. [ibireme大神文章](https://blog.ibireme.com/2015/11/12/smooth_user_interfaces_for_ios/)