---
layout: post
title: "[UIView viewWithTag:]"
date: 2014-11-11 01:46:01 +0800
categories: [uiview, objective-c, ios]
keywords: "viewWithTag, UIView, Objectice-C, iOS, UIKit"
description: "[UIView viewWithTag:] 觀念"
---
這篇關於使用 [UIView viewWithTag:] 的文章，雖然已是兩年前的，但在下認為文章內所提及的觀念與用法還是很值得與大家分享。

UIView 有個 tag 屬性以及相對應的方法 -viewWithTag:，讓我們可以輕易的存取特定的 view，而不需要額外的 reference。

這邊作者分享了幾個使用上常見的問題：

* 不要使用 tags 來儲存資料，例如：陣列存取物件的索引 (index)...，因為這是一件容易造成後續維護者難以理解你的 code 的一件事...底下是一個使用 button 來呈現圖片縮圖的例子，為了記得哪個按鈕被按到以及要呈現哪個對應於 array 中的圖片索引，所以利用 tag 來記錄。~~(但其實還滿容易這麼幹的啊...在下也常這麼做，要好好檢討了orz)~~

		- (void)configureThumbnailButton:(UIButton *)thumbnailButton 
		                        forPhoto:(NSUInteger)photoIndex {
    		// ...
	    	thumbnailButton.tag = 1 + photoIndex;
    		// ...
		}

		- (void)thumbnailButtonTapped:(id)sender {
    		UIButton *thumbnailButton = sender;
    		NSUInteger photoIndex = thumbnailButton.tag - 1;
    		id selectedPhoto = [self.photoArray objectAtIndex:photoIndex];
    		[self showPhoto:selectedPhoto];
		}
		
* 存取 view 額外資料的幾個比較正確的方式：

  我們來探討一下為什麼會寫出上述例子的程式碼，其實也只是想要知道 button 對應的 index，或是 button 真正對應的圖片縮圖，此時更適合的方式應該是另外做一個 custom subClass 來儲存這些有用的額外資料，延續上述例子，應該可以這麼做：
  		
		@interface PhotoThumbnailButton : UIButton

		@property (nonatomic, assign) NSInteger photoIndex;
		// OR
		@property (nonatomic, strong) Photo *photo;

		@end
		
  這時候你會說，如果每次都要這樣也太麻煩了吧！的確，每次都要建個 subClass 有時可能不符合實際情況，底下也提供一個用法：Associated references。
  
* [Associated references](http://goo.gl/6fM6Kq)：提供了 objc_getAssociatedObject 以及 objc_setAssociatedObject 等方法，可以利用特定的 key 及 policy 存取特定的 obejct。如：

		#import <objc/runtime.h>

		static char kThumbnailButtonAssociatedPhotoKey;

		// ...

		- (void)setAssociatedPhoto:(Photo *)associatedPhoto
		        forThumbnailButton:(UIButton *)thumbnailButton {
    		objc_setAssociatedObject(thumbnailButton,
    		                         &kThumbnailButtonAssociatedPhotoKey,
                                     associatedPhoto,
                                     OBJC_ASSOCIATION_RETAIN_NONATOMIC);
		}

		- (Photo *)associatedPhotoForThumbnailButton:(UIButton *)thumbnailButton {
			return objc_getAssociatedObject(thumbnailButton,
                                    		 &kThumbnailButtonAssociatedPhotoKey);
		}
		
  存取時，
  
  		- (void)configureThumbnailButtonForPhoto:(Photo *)photo {
    		// ...
    		[self setAssociatedPhoto:photo forThumbnailButton:thumbnailButton];
    		// ...
		}

		- (void)thumbnailButtonTapped {
    		Photo *photo = [self associatedPhotoForThumbnailButton:thumbnailButton];
    		// ...
		}
		
  真的是很酷啊... 可是，如果只是想要快速直接的知道是哪一個 view，這樣的方式還是不夠直覺...所以，有些情況下，你還是會想直接利用 UIView's tag 存取，但重點是請遵循 UIView's tag 是 UIView 的唯一識別、可讀性以及易維護性的觀念，而使用 #define 或 enum value 來設定 tags，在你之後的開發者會很感謝你的..XD，例如：
  
		enum MyViewTags {
			kTitleLabelTag = 1,
    		kSendButtonTag,
    		kSomeOtherViewTag
		};

		// ...

		if (sender.tag == kSendButtonTag) {
    		// ...
		}

* 結論：
  * 不要使用 tags 來儲存資料 -> use sub-class or associated references resolving this.
  * 多利用 public properties of sub-class for getting references of subviews.
  * If you do use tags, do not use magic numbers. Use named constants.
  
* [Ref.](http://goo.gl/0xHqWt)



