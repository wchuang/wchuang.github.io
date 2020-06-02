---
layout: post
title: "Distribute an iOS application OTA"
date: 2015-04-08 00:53:56 +0800
categories: [iOS, OTA]
keywords: "iOS, OTA, AdHoc, distribute"
description: "建立可給外部使用者測試的 AdHoc iOS app OTA"
---

簡單記錄如何建立給外部使用者或客戶測試的 AdHoc 版本 

iOS app OTA (over the air)，

而如何產生憑證及 archive 出 AdHoc 就不再贅述了，網路也有很多資源。

1. 重要的是描述下載位置的 HTML、描述 app 的 plist 檔案以及 app 的 .ipa 檔案，找台伺服器來存放它們吧！這邊推薦使用 Dropbox 方便又好用，當然你也可以用自家 server 或其它 web storage services。

2. .ipa 檔：AdHoc 出 .ipa 檔案，並上傳到 Dropbox 底下，點選分享後，會拿到一個分享連結，如 https://www.dropbox.com/s/ooxxaabbcc/ForTestingApp.ipa?dl=0 ，把 www.dropbox.com 改成 dl.dropboxusercontent.com 及 ?dl=0 拿掉，會變成 -> https://dl.dropboxusercontent.com/s/ooxxaabbcc/ForTestingApp.ipa 。

3. plist 檔：加入一個 Array 的 key，名稱取為 assets，負責描述 .ipa、app icon 兩張（57 * 57）及 (512 * 512) 的分享連結路徑 (與上述 .ipa 檔案ㄧ樣，需放在 dropbox 中，並開啟分享取得連結)，如果不放圖片的話，下載中就看不到 app 的 icon 囉。再加入一個 Dictionary 的 key，名稱取為 metadata，描述 app 相關資訊，如 bundle-identifier、title 或 subtitle 等等。最後一樣開啟分享取得連結。

4. HTML 檔：最後是 HTML，拿來記錄上述 plist 檔的分享位置，並做個畫面呈現給使用者點選下載，可以簡單放上下列程式。

    	<a href="itms-services://?action=download-manifest&url=
    	https://dl.dropboxusercontent.com/s/ooxxaabbcc/ForTestingApp.plist">
    	Install this awesome app!</a>
    	
完成後你就可以把 HTML 的連結給你的朋友或客戶測試囉，不過要記得把手機的 UDID 加進去 Provisioning Profile。

以上，有不清楚的地方歡迎指教 :p



