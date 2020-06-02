---
layout: post
title: "如何使用 @autoclosure 設計 Swift APIs"
date: 2019-03-02T16:31:08+08:00
comments: true
categories: [swift, ios]
keywords: "aotuclosure, Swift, iOS"
---

Swift 中的 `@autoclosure` 屬性，可以讓你定義一個 `可自動取得一個閉包 (closure) 內的包裹 (wrapped)` 的參數。它主要用於延緩執行時間，而不是當這個參數傳入的時候就直接使用，因為執行的動作可能有點耗時。這麼說有點繞口，下面是一個官方 Swift 標準函式庫的 `assert` 例子，我們來看看 Apple 是怎麼實現這個方法的。

由於 `assert` 只有在 `debug` 模式下才會被觸發，在 `release` 模式下會被返回而不會執行 `expression`，完整的實作請看源碼: [Assert.swift](https://github.com/apple/swift/blob/master/stdlib/public/core/Assert.swift)

```
func assert(_ expression: @autoclosure () -> Bool, 
			_ message: @autoclosure () -> String) {
	guard let isDebug else { return }

	if !expression() {
		assertionFailure(message())
	}
}
```

如果 `assert` 的實作方式是用一般的閉包實現，那你會這樣呼叫：

```
assert({ someCondition() }, { "hey, it failed." })
```

現在你只需要這樣即可，不用在傳入一個閉包參照。

```
assert(someCondition(), "hey, it failed.")
```

那麼如何運用 `@autoclosure` 這個方便的特性在實際 APIs 設計上面呢？

###1. Inlining assignments 

在方法呼叫的時候，把一個表達式當做一個參數傳入。一般在 iOS 處理畫面動畫你會用到：

```
UIView.animate(withDuration: 0.25) {
	view.frame.origin.y = 100
}
```

在 `@autoclosure` 中，我們定義一個方法來化簡這樣的呼叫方式。

```
func animate(_ animation: @autoclosure @escaping () -> Void, 
				duration: TimeInterval = 0.25) {
	UIView.animate(withDuration: duration, animations: animation)
}
```

如此一來，我們只需要傳入一個特定操作表達式，不用再寫額外的 `{}`。
增加了程式碼的可讀性以及降低複雜度。

```
animate(view.frame.origin.y = 100)
```

### 2. Passing errors as expressions

我們常常會需要寫一個工具來處理 errors 的各種情況，此時使用 `@autoclosure` 就變得非常方便。
例如：我們寫一個 `extension` 來擴展 `Optional`，讓我們可以容易的解包 (unwrap) 或拋出異常 (throwing API)。

```
extension Optional {
	func unwrapOrThrow(_ errorExpression: @autoclosure () -> Error) throws -> Wrapped {
		guard let value = self else {
			throw errorExpression()
		}
		return value
	}
}
```

這跟上述的 `assert` 一樣，我們只需要關注在錯誤的表達式上面，而不需要每次都去判斷是否解包成功 (unwarp)。
我們現在可以這樣使用 `unwarpOrThrow` API：

```
let name = try argument(at: 1).unwarpOrThrow(ArgumentError.missingName)
```

### 3. Type inference using default values

很多時候當我們從 `Dictionary`, `TextField.text`, `UserDefaults` 或者其他地方取值的時候，常常會碰到可選值 (optional value) 的情況。

如：

```
let coins = (dictionary["numberOfCoins"] as? Int) ?? 100
```

這樣的寫法用了很多操作符號，程式碼變得難以閱讀。
這時候可以用 `@aotuclosure` 來改善寫法。

```
let coins = dictioanry.value(forKey: "numberOfCoins", defaultValue: 100)
```

這邊賦予了一個預設值給 `coins`，當內容為 nil 的情況。這樣是不是好閱讀多了，也不需要每次都去判斷是否為 nil。:grin:
不過我們還需要一個方法擴展 Dictionary。

```
extension Dictionary where Value == Any {
	func value<T>(forKey key: Key, defaultValue: @autoclosure () -> T) -> T {
		guard let value = self[key] as? T else { return defaultValue() }
		return value
	}
}
```

### 結論

`@autoclosure` 是一個很有用的工具，讓你可以關注在你想處理的事情上面，並減少了重複性判斷，大大增加了程式碼的可讀性。


### Ref. 

[swiftbysundell](https://www.swiftbysundell.com/posts/using-autoclosure-when-designing-swift-apis)











