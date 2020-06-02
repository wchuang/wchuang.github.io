---
layout: post
title: "PresentViewController delay on UITableViewCell"
date: 2015-11-22 02:36:24 +0800
categories: [ios, uiview]
keywords: "iOS, UITableView, UITableViewCell, presentViewController"
description: "presenting view controller delay on UITableViewCell"
---

最近開發常常遇到：當點選 UITableViewCell 時去呼叫 presentViewController:animated:completion:，欲開啟的 view controller 會延遲出現的問題，這個似乎是 iOS 7 的 bug，不過之前開發手機卻沒什麼遇到，直到最近開發 iPad 才發現。

原因是當 UITableViewCell 的 selectionStyle 設為 UITableViewCellSelectionStyleNone，也就是希望不要出現點選效果時，造成沒有動畫效果去觸發 main runloop，所以 thread 好像是睡著了...

解決方式：

在呼叫 presentViewController:animated:completion: 後

* 呼叫 CFRunLoopWakeUp(CFRunLoopGetCurrent())
* 或用 GCD 在 main queue 執行一個空的 block：dispatch_async(dispatch_get_main_queue(), ^{})
* 或用 GCD 在 main queue 直接呼叫 presentViewController:animated:completion:

From: [Stackoverflow 討論](http://stackoverflow.com/questions/21075540/presentviewcontrolleranimatedyes-view-will-not-appear-until-user-taps-again), [Thread run loops](https://developer.apple.com/library/ios/documentation/Cocoa/Conceptual/Multithreading/RunLoopManagement/RunLoopManagement.html)
