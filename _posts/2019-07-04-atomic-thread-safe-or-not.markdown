---
layout: post
title: "Atomic 原理與線程安全"
date: 2019-07-04T18:50:03+08:00
categories: [iOS, atomic, thread]
keywords: "iOS, atomic, property, thread-safe"
---

## 寫在前面

這篇僅研究 iOS 中 `atomic` 屬性底層源碼實現與線程安全的關係。至於 atomic 的介紹以及與 nonatomic 的比較就不在這邊展開了。

## Atomic 能保證線程安全嗎？

首先，我們看下線程安全在 Wiki 的定義：

```
執行緒安全是編程中的術語，指某個函數、函數庫在多執行緒環境中被調用時，
能夠正確地處理多個執行緒之間的共享變量，使程序功能正確完成。 
```

也就是說，在你的程序中，若含有多個執行緒同時的執行某一段任務，在 A 執行緒執行完成後的結果不應該被 B 執行緒的結果所影響。這就可以被稱作是線程安全的，即能夠**保證資料的一致性及完整性**。

在回答這個問題之前，我們先理解一下 atomic 在底層是怎麼實現的。

## Atomic 源碼實現

**Set value**

```
if (!atomic) {
    oldValue = *slot;
    *slot = newValue;
} else {
    spinlock_t& slotlock = PropertyLocks[slot];
    slotlock.lock();
    oldValue = *slot;
    *slot = newValue;        
    slotlock.unlock();
}
```

**Get value**

```
spinlock_t& slotlock = PropertyLocks[slot];
slotlock.lock();
id value = objc_retain(*slot);
slotlock.unlock();
```    

在 Apple 官方開源的 objc4-750 裡面，可以發現在屬性在讀寫的時候，進行了 atomic 的判斷式，如果是 atomic 的屬性，會以該屬性的記憶體位置當作 key 值去底層維護的 mapping 表找出對應的 `spinlock`，並且加鎖，待完成任務後才解鎖。

所以，我們知道了，**atomic 會利用 spinlock 自旋鎖來確保讀寫過程中不被其他的執行緒篡改內容**，算是線程安全。

所以我寫了一個例子來測試是否如我們想的那樣呢。

```
dispatch_queue_t concurrent = dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0);

    // Thread A
    dispatch_async(concurrent, ^{
        // Task A
        NSLog(@"Enter Thread A");
        for (int count = 0; count < 100; count++) {
            self.atomicStr = @"A";
            self.nonAtomicStr = @"A";
            if (![@"A" isEqualToString:_atomicStr]) {
                NSLog(@"[Atomic] Task A's result is not expected. %@", _atomicStr);
            }
            if (![@"A" isEqualToString:_nonAtomicStr]) {
                NSLog(@"[NonAtomic] Task A's result is not expected. %@", _nonAtomicStr);
            }
        }
    });
    
    // Thread B
    dispatch_async(concurrent, ^{
        // Task B
        NSLog(@"Enter Thread B");
        for (int count = 0; count < 100; count++) {
            self.atomicStr = @"B";
            self.nonAtomicStr = @"B";
            if (![@"B" isEqualToString:_atomicStr]) {
                NSLog(@"[Atomic] Task B's result is not expected. %@", _atomicStr);
            }
            if (![@"B" isEqualToString:_nonAtomicStr]) {
                NSLog(@"[NonAtomic] Task B's result is not expected. %@", _nonAtomicStr);
            }
        }
    });
```

這裡我開了兩個線程同時分別修改 `atomic` 及 `nonatomic` 字串。因為我們已經知道 atomic 在賦值的時候會加鎖，來保證當前線程下的任務不會被其他線程修改。所以，我們預期 Task A 每次賦值後的結果要為 A，Task B 需為 B。

```
Atomic[43745:2218609] Enter Thread A
Atomic[43745:2218674] Enter Thread B
Atomic[43745:2218609] [Atomic] Task A's result is not expected. B
Atomic[43745:2218674] [NonAtomic] Task B's result is not expected. A
Atomic[43745:2218674] Enter Thread A
Atomic[43745:2218609] Enter Thread B
Atomic[43745:2218674] [Atomic] Task A's result is not expected. B
Atomic[43745:2218674] [NonAtomic] Task A's result is not expected. B
```

但卻出現這樣的結果。**A 與 B 任務還是會互相影響。宣告 atomic 與 nonatomic 結果是一樣的**。

所以 atomic 不是線程安全嗎？

這邊我的理解是，atomic 是安全的，但是它的職責只在於**保證多執行緒下對同個屬性的讀寫不會同時進行**。所以當 A 開始寫入，B 的寫入會進入等待，等到 A 一寫完鎖釋放後，B 馬上又鎖上進行寫的操作。這時候 A 的讀取又會進入等待，等到 B 寫完鎖釋放了之後，這時候在 Task A 裡面就會讀取到 B 的內容。反之，如果 A 的讀取快於 B 的寫入，那麼，A 讀取到的內容就會是預期的 A。至於**執行緒的執行順序得看系統的調度了**。

## 如何實現完整的線程安全？

其實就是確保單一任務或接口方法的原子性，讓某一執行緒在執行某段任務或某個方法的時候是唯一的。
上述例子為了方便測試我加上了 `@synchronized` 鎖。這樣就是預期的結果了。

如下：

```
// Thread A
    dispatch_async(concurrent, ^{
        // Task A
        NSLog(@"Enter Thread A");
        @synchronized (self) {
            for (int count = 0; count < 100; count++) {
                self.atomicStr = @"A";
                self.nonAtomicStr = @"A";
                if (![@"A" isEqualToString:_atomicStr]) {
                    NSLog(@"[Atomic] Task A's result is not expected. %@", _atomicStr);
                }
                if (![@"A" isEqualToString:_nonAtomicStr]) {
                    NSLog(@"[NonAtomic] Task A's result is not expected. %@", _nonAtomicStr);
                }
            }
        }
    });
```

完整範例請看 -> [AtomicTest](https://github.com/wchuang/devSwift/tree/master/Atomic)

## 結論

**Atomic 是安全的，它保證了多執行緒下，讀和寫同時間的操作只能有一個，也就是資料的唯一性**。只是它無法確保是哪個線程先執行，才造成了最後的結果不一致。所以，還需要對每個線程裡的任務做鎖的保護，確保最後的結果是預期完整的。

Enjoy!
