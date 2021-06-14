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

## AOP v.s. OOP

Aspects 是 iOS 中實現 [AOP 設計模式](https://zh.wikipedia.org/wiki/%E9%9D%A2%E5%90%91%E5%88%87%E9%9D%A2%E7%9A%84%E7%A8%8B%E5%BA%8F%E8%AE%BE%E8%AE%A1)的一種實現。

OOP 注重的是對象的屬性、行為的封裝，AOP 專注於某個重複的處理步驟或運算邏輯，從中進行切面的擷取，以降低耦合度。

比如：你有一個 event tracking 的需求，以 OOP 來說你可能會運用一些封裝方法，在每個方法都加上這個 tracking 方法，但是以 AOP 來看，就是把這些 tracking 的方法提取出來，運用動態特性，實現這些程式碼耦合，OOP 及 AOP 不是互斥而是需要配合。

在 iOS 裡面使用 AOP 可以實現無代碼入侵，主要可以用於一些具有橫向（跨模塊）的服務，如：Logger、Event tracking、Cache 等等。

## NSMethodSignature & NSInvocation

由於 Aspects 運用到了 iOS runtime 的 method swizzling，

在 iOS 中調用方法除了

1. 一般方式：```[某個類的實例 方法名: 參數];```
2. PerformSelector 系列：```[某個類的實例 performSelector: @selector(方法:) withObject: 參數];```

還能利用 NSMethodSignature 以及 NSInvocation 完成方法調用。

在 Aspects 中也用到了。這裡先來複習一下。

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

舉個栗子：

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

至於找不到方法的處理方式，網上很多資源，這裡就不多說了，

直接來看下 Aspects 的源碼吧。

## Aspects 內部結構

### _AspectBlock

```
typedef struct _AspectBlock {
	__unused Class isa;
	AspectBlockFlags flags;
	__unused int reserved;
	void (__unused *invoke)(struct _AspectBlock *block, ...);
	struct {
		unsigned long int reserved;
		unsigned long int size;
		// requires AspectBlockFlagsHasCopyDisposeHelpers
		void (*copy)(void *dst, const void *src);
		void (*dispose)(const void *);
		// requires AspectBlockFlagsHasSignature
		const char *signature;
		const char *layout;
	} *descriptor;
	// imported variables
} *AspectBlockRef;
```


### AspectInfo

> **先說結論：這個類主要紀錄 NSInvocation 訊息，它將 NSInvocation 包裝一層，如：所有參數訊息、及原本的 NSInvocation**

Aspects 在 .h 檔案定義了 AspectInfo protocol，在 hook 方法的時候會傳入一 block，而這個 block 必須遵守這個協議。

```
/// The AspectInfo protocol is the first parameter of our block syntax.
@protocol AspectInfo <NSObject>

/// The instance that is currently hooked.
- (id)instance;

/// The original invocation of the hooked method.
- (NSInvocation *)originalInvocation;

/// All method arguments, boxed. This is lazily evaluated.
- (NSArray *)arguments;

@end
```

接著，Aspects 在 .m 檔案上，
聲明了繼承 NSObject 的 AspectInfo 類，並實現 AspectInfo 協議。

```
@interface AspectInfo : NSObject <AspectInfo>
- (id)initWithInstance:(__unsafe_unretained id)instance invocation:(NSInvocation *)invocation;
@property (nonatomic, unsafe_unretained, readonly) id instance;
@property (nonatomic, strong, readonly) NSArray *arguments;
@property (nonatomic, strong, readonly) NSInvocation *originalInvocation;
@end
```

它的對應實現

把外面傳進來的對象以及原始的 invocation 保存至 AspectInfo。

arguments 是一個懶加載方法，返回的是原始 invocation 的 aspects_arguments array。

aspects_arguments 的實現是 NSInvocation 的 Aspects category 擴展（見下面）。

```
@implementation AspectInfo

@synthesize arguments = _arguments;

- (id)initWithInstance:(__unsafe_unretained id)instance invocation:(NSInvocation *)invocation {
    NSCParameterAssert(instance);
    NSCParameterAssert(invocation);
    if (self = [super init]) {
        _instance = instance;
        _originalInvocation = invocation;
    }
    return self;
}

- (NSArray *)arguments {
    // Lazily evaluate arguments, boxing is expensive.
    if (!_arguments) {
        _arguments = self.originalInvocation.aspects_arguments;
    }
    return _arguments;
}

@end
```
### NSInvocation (Aspects)

aspects_arguments 返回該 NSInvocation 中的所有參數。

```
@interface NSInvocation (Aspects)
- (NSArray *)aspects_arguments;
@end
```

這裡的 for-loop 是從 index 2 開始，然後呼叫了 `aspect_argumentAtIndex` 自定義方法返回方法簽名裡面指定 index 的 type encoding string。

```
- (NSArray *)aspects_arguments {
	NSMutableArray *argumentsArray = [NSMutableArray array];
	for (NSUInteger idx = 2; idx < self.methodSignature.numberOfArguments; idx++) {
		[argumentsArray addObject:[self aspect_argumentAtIndex:idx] ?: NSNull.null];
	}
	return [argumentsArray copy];
}
```

所以這邊有兩個研究點：

1. 為什麼 index 從 2 開始：
	根據上述的 `@encode()` type encoding，會以 "返回值 type + 參數 types" 組合的編碼，還有另外的隱含參數：`self` 以及 `_cmd`。
	
	如：

	```
		- (void)tap; => "v@:"
		- (int)tapWithView:(double)pointx; => "i@:d"
	```
	
	可知：前三位代表 `返回值 + @(self) + :(_cmd)`，第一位返回值不在 arguments 的計數範圍，所以扣掉第一位返回值後，從第三位開始才是傳入的參數（index = 2 的位置）。
	
	Function   | return value  | 0 | 1 | 2
	--------------|:-----:|-----:| ----:|------------------------
	tap    | v |  @ | : | N/A
	tapWithView | v |  @ |  : | d

	所以從 index 2 開始才能得到我們要的所有參數。
	
2. `aspect_argumentAtIndex` 實現原理：
	
	這裡有借鏡了 `ReactiveCocoa` 的取得方法簽名參數的實現方法
	
	```
	// Thanks to the ReactiveCocoa team for providing a generic solution for this.
	- (id)aspect_argumentAtIndex:(NSUInteger)index {

	const char *argType = [self.methodSignature getArgumentTypeAtIndex:index];

	// Skip const type qualifier.
	if (argType[0] == _C_CONST) argType++;

	#define WRAP_AND_RETURN(type) do { type val = 0; [self getArgument:&val atIndex:(NSInteger)index]; return @(val); } while (0)

	if (strcmp(argType, @encode(id)) == 0 || strcmp(argType, @encode(Class)) == 0) {
		__autoreleasing id returnObj;
		[self getArgument:&returnObj atIndex:(NSInteger)index];
		return returnObj;
	} else if (strcmp(argType, @encode(SEL)) == 0) {
        SEL selector = 0;
        [self getArgument:&selector atIndex:(NSInteger)index];
        return NSStringFromSelector(selector);
    } else if (strcmp(argType, @encode(Class)) == 0) {
        __autoreleasing Class theClass = Nil;
        [self getArgument:&theClass atIndex:(NSInteger)index];
        return theClass;
        // Using this list will box the number with the appropriate constructor, instead of the generic NSValue.
	} else if (strcmp(argType, @encode(char)) == 0) {
		WRAP_AND_RETURN(char);
	} else if (strcmp(argType, @encode(int)) == 0) {
		WRAP_AND_RETURN(int);
	} else if (strcmp(argType, @encode(short)) == 0) {
		WRAP_AND_RETURN(short);
	} else if (strcmp(argType, @encode(long)) == 0) {
		WRAP_AND_RETURN(long);
	} else if (strcmp(argType, @encode(long long)) == 0) {
		WRAP_AND_RETURN(long long);
	} else if (strcmp(argType, @encode(unsigned char)) == 0) {
		WRAP_AND_RETURN(unsigned char);
	} else if (strcmp(argType, @encode(unsigned int)) == 0) {
		WRAP_AND_RETURN(unsigned int);
	} else if (strcmp(argType, @encode(unsigned short)) == 0) {
		WRAP_AND_RETURN(unsigned short);
	} else if (strcmp(argType, @encode(unsigned long)) == 0) {
		WRAP_AND_RETURN(unsigned long);
	} else if (strcmp(argType, @encode(unsigned long long)) == 0) {
		WRAP_AND_RETURN(unsigned long long);
	} else if (strcmp(argType, @encode(float)) == 0) {
		WRAP_AND_RETURN(float);
	} else if (strcmp(argType, @encode(double)) == 0) {
		WRAP_AND_RETURN(double);
	} else if (strcmp(argType, @encode(BOOL)) == 0) {
		WRAP_AND_RETURN(BOOL);
	} else if (strcmp(argType, @encode(bool)) == 0) {
		WRAP_AND_RETURN(BOOL);
	} else if (strcmp(argType, @encode(char *)) == 0) {
		WRAP_AND_RETURN(const char *);
	} else if (strcmp(argType, @encode(void (^)(void))) == 0) {
		__unsafe_unretained id block = nil;
		[self getArgument:&block atIndex:(NSInteger)index];
		return [block copy];
	} else {
		NSUInteger valueSize = 0;
		NSGetSizeAndAlignment(argType, &valueSize, NULL);

		unsigned char valueBytes[valueSize];
		[self getArgument:valueBytes atIndex:(NSInteger)index];

		return [NSValue valueWithBytes:valueBytes objCType:argType];
	}
	return nil;
	#undef WRAP_AND_RETURN
	}
	```
	
	`strcmp` 這個方法是判斷我們從方法簽名裡面取出來的 type encoding 是否跟 `@encode(id)` 的 type encoding 是一樣的，如果返回 0，則表示類型是相同的。
	
	所以就依序根據各種類型比較判斷。基本數據類型就會依據 `WRAP_AND_RETURN(type)` 宏定義的方法呼叫 NSInvocation 中的 `getArgument:atIndex:` 方法，用來取得對應的參數，最後返回 `@(val)` Number 對象。
	
	最後，block 及 struct 也會返回對應的對象。

### AspectIdentifier

```
// Tracks a single aspect.
@interface AspectIdentifier : NSObject

+ (instancetype)identifierWithSelector:(SEL)selector object:(id)object options:(AspectOptions)options block:(id)block error:(NSError **)error;

- (BOOL)invokeWithInfo:(id<AspectInfo>)info;

@property (nonatomic, assign) SEL selector;
@property (nonatomic, strong) id block;
@property (nonatomic, strong) NSMethodSignature *blockSignature;
@property (nonatomic, weak) id object;
@property (nonatomic, assign) AspectOptions options;

@end
```

它其實就是 Aspects 用來描述某一個切片的一個封裝類，類裡面記錄了切片需要的信息，如：原本類的 selector，你想要 hook 的 block 方法，block 的方法簽名，原本類或實例對象，以及 AspectOptions 調用時機。

以及兩個方法（下文會介紹）：

1. 產生 AspectIdentifier 類方法
2. 切片調用方法

### AspectContainer

```
// Tracks all aspects for an object/class.
@interface AspectsContainer : NSObject

- (void)addAspect:(AspectIdentifier *)aspect withOptions:(AspectOptions)injectPosition;

- (BOOL)removeAspect:(id)aspect;

- (BOOL)hasAspects;

@property (atomic, copy) NSArray *beforeAspects;
@property (atomic, copy) NSArray *insteadAspects;
@property (atomic, copy) NSArray *afterAspects;

@end

```

一個紀錄所有 AspectIdentifier 的容器，包含了三個 Array，分別紀錄前調用、完全取代、以及後調用的 AspectIdentifier。

### AspectTracker

```
@interface AspectTracker : NSObject

- (id)initWithTrackedClass:(Class)trackedClass;

@property (nonatomic, strong) Class trackedClass;
@property (nonatomic, readonly) NSString *trackedClassName;
@property (nonatomic, strong) NSMutableSet *selectorNames;
@property (nonatomic, strong) NSMutableDictionary *selectorNamesToSubclassTrackers;

- (void)addSubclassTracker:(AspectTracker *)subclassTracker hookingSelectorName:(NSString *)selectorName;
- (void)removeSubclassTracker:(AspectTracker *)subclassTracker hookingSelectorName:(NSString *)selectorName;
- (BOOL)subclassHasHookedSelectorName:(NSString *)selectorName;
- (NSSet *)subclassTrackersHookingSelectorName:(NSString *)selectorName;

@end
```

AspectTracker 用來追蹤你要 hook 的類，trackedClass 是你要 hook 的類，trackedClassName 是要 hook 的類名，selectorNames 紀錄所有要被 hook 的 selector name，selectorNamesToSubclassTrackers 是一個字典，key 為 selector name，value 為一組 AspectTracker set。

## Aspects hook 流程

1. 利用 NSObject category，增加了兩個 hook 方法，類方法以及實例方法。所以只要是 NSObject 類都可以直接使用下面兩個方法。

	方法需要傳入四個參數，作者在註釋上都說得滿清楚的
	
	1. 原方法的 selector
	2. AspectOptions，可選 before、instead 或者 after
	3. 遵循 AspectInfo protocol 的 Block，Aspects 會複製被 hook 的原方法的方法簽名
	4. 返回的錯誤訊息，如果有的話

	返回值是 AspectToken 可以用來取消這個 hook 操作。
	
	註：Aspects 不能 hook 靜態方法。
	
	```
	/// Adds a block of code before/instead/after the current `selector` for a specific class.
	///
	/// @param block Aspects replicates the type signature of the method being hooked.
	/// The first parameter will be `id<AspectInfo>`, followed by all parameters of the method.
	/// These parameters are optional and will be filled to match the block signature.
	/// You can even use an empty block, or one that simple gets `id<AspectInfo>`.
	///
	/// @note Hooking static methods is not supported.
	/// @return A token which allows to later deregister the aspect.
	+ (id<AspectToken>)aspect_hookSelector:(SEL)selector
	                           withOptions:(AspectOptions)options
	                            usingBlock:(id)block
	                                 error:(NSError **)error;
	
	/// Adds a block of code before/instead/after the current `selector` for a specific instance.
	- (id<AspectToken>)aspect_hookSelector:(SEL)selector
	                           withOptions:(AspectOptions)options
	                            usingBlock:(id)block
	                                 error:(NSError **)error;
	```
	
	這裡滿有趣的，作者提到
	
	> Aspects uses Objective-C message forwarding to hook into messages. This will create some overhead. Don't add aspects to methods that are called a lot. Aspects is meant for view/controller code that is not called a 1000 times per second.
	 
	> Adding aspects returns an opaque token which can be used to deregister again. All calls are thread safe.
	
	Aspects 利用了 Objective-C 的消息轉發機制 hook 消息，這樣會帶來性能上的消耗。所以不要把它用在會被呼叫很多次的方法，它是設計給 View/ViewController 使用，不要用在一秒被呼叫1000次的方法。XD
	
	且加完 Aspects 後，會返回一個 token，這個 token 可以拿來註銷 hook 方法。而所有的呼叫都是線程安全的（用到了自旋鎖。

2. 這兩個方法其實內部都是調用了同一個私有方法 aspect_add

	```
	static id aspect_add(id self, SEL selector, AspectOptions options, id block, NSError **error) {
	    NSCParameterAssert(self);
	    NSCParameterAssert(selector);
	    NSCParameterAssert(block);
	
	    __block AspectIdentifier *identifier = nil;
	    aspect_performLocked(^{
	        if (aspect_isSelectorAllowedAndTrack(self, selector, options, error)) {
	            AspectsContainer *aspectContainer = aspect_getContainerForObject(self, selector);
	            identifier = [AspectIdentifier identifierWithSelector:selector object:self options:options block:block error:error];
	            if (identifier) {
	                [aspectContainer addAspect:identifier withOptions:options];
	
	                // Modify the class to allow message interception.
	                aspect_prepareClassAndHookSelector(self, selector, error);
	            }
	        }
	    });
	    return identifier;
	}
	```

	這個方法有五個參數，

	1. self：自己
	2. selector：你需要 hook 的 SEL
	3. options：定義的 AspectOptions，你可選擇 hook 方法被調用的時機，如：AspectPositionAfter（之後調，這個是預設）、AspectPositionInstead（完全取代原方法，比較危險） 、AspectPositionBefore（之前調）或 AspectOptionAutomaticRemoval（只執行一次）
	4. block：你要執行的方法
	5. error：如果發生錯誤

	一開始對必要參數做檢查，接著利用了自定義的自旋鎖來保證線程安全，這個鎖對於少量代碼的操作性能較高，這裡就先不討論自旋鎖優先級反轉的問題。

	```
	static void aspect_performLocked(dispatch_block_t block) {
	    static OSSpinLock aspect_lock = OS_SPINLOCK_INIT;
	    OSSpinLockLock(&aspect_lock);
	    block();
	    OSSpinLockUnlock(&aspect_lock);
	}
	```

	接著，檢查這個方法是否可以被 hook，具體代碼實現比較長就不貼出來了，感興趣可以下載源碼來看一下，這段主要做了這些事情：

	1. 建立黑名單：把系統方法，retain/release/autorelease/forwardInvocation 加入黑名單，不允許 hook 這些方法
	2. 如果你 hook 的是 dealloc，則 options 只能是 AspectPositionBefore
	3. Hook 對象必須有實現這個方法
	4. 如果你 hook 元類 (meta-class) 方法的話，則會多判斷：
		* 先取得全局的字典，裡面記錄被 hook 的類以及對應的 AspectTracker 對象。這個 tracker 記錄著 hook 的類，類名，所有被 hook 的方法名，以及 selectorNamesToSubclassTrackers 是個字典，記錄著某個方法名（key），對應的所有子類的 AspectTracker（value）。所以當它從 selectorNamesToSubclassTrackers 裡面檢查到某個方法已經有 AspectTracker，則表示已經有它的子類 hook 過這個方法了，此時便不允許再次 hook。
		* 接著，用 for-loop 遍歷所有父類，如果 tracker 紀錄的所有被 hook 的方法名已經有了，則不允許再次 hook
		* 如果上述都不滿足，即所有子類和父類都沒有重複 hook，則產生一個 AspectTracker 對象，然後把 hook 相關信息紀錄起來，子類存完同步到父類，然後不斷的往父類遍歷直到 root class(NSObject)

	以上是檢查方法是否可以被 hook。
	
	如果都通過，開始進入 hook 方法存儲流程，
	
	首先要產生 AspectsContainer，這裡利用了關聯對象創建或者取出 AspectsContainer，接著產生 AspectIdentifier，AspectIdentifier 紀錄了 selector 方法，block 執行回調，block 方法簽名，options，以及一個 weak object 指向了自己（調用對象）。
	
	那是怎麼產生 AspectIdentifier 呢？透過封裝的 AspectIdentifier 類方法

	```
	+ (instancetype)identifierWithSelector:(SEL)selector object:(id)object options:(AspectOptions)options block:(id)block error:(NSError **)error {
	    NSCParameterAssert(block);
	    NSCParameterAssert(selector);
	    
	    NSMethodSignature *blockSignature = aspect_blockMethodSignature(block, error); // TODO: check signature compatibility, etc.
	    if (!aspect_isCompatibleBlockSignature(blockSignature, object, selector, error)) {
	        return nil;
	    }
	
	    AspectIdentifier *identifier = nil;
	    if (blockSignature) {
	        identifier = [AspectIdentifier new];
	        identifier.selector = selector;
	        identifier.block = block;
	        identifier.blockSignature = blockSignature;
	        identifier.options = options;
	        identifier.object = object; // weak
	    }
	    return identifier;
	}
	```
	
	這裡需要先拿到 block 的方法簽名，拿到後檢查跟原有對象的方法簽名是否一致。因為你傳進來 block 裡面放著你想要做的事情，為了方便方法交換後的 block 方法調用，所以需要轉為方法簽名，也是文章一開始提到過的方法調用的技巧。
	
	這裡轉換用了文章一開始提到的 AspectBlockRef 結構體，將 block bridge 到這結構體，然後把方法描述轉為方法簽名，這個結構體是模仿了 block 的內存佈局。
	
	```
	const char *signature = (*(const char **)desc);
	return [NSMethodSignature signatureWithObjCTypes:signature];
	```
	拿到 AspectIdentifier 後，加入 AspectsContainer。（AspectIdentifier 可以視為是每個 Aspect 的基本信息，比如 hook 的方法，要執行的 block 方法簽名，原 block 等）
	
	AspectsContainer 裡面有三個 array，分別紀錄先調用，後調用或者完全取代調用的 AspectIdentifier。
	
	如果都完成了信息紀錄，則開始進入 hook 流程。
	
3. 準備 Hook 流程

	```
	static void aspect_prepareClassAndHookSelector(NSObject *self, SEL selector, NSError **error) {
    NSCParameterAssert(selector);
    
	    Class klass = aspect_hookClass(self, error);
	    Method targetMethod = class_getInstanceMethod(klass, selector);
	    IMP targetMethodIMP = method_getImplementation(targetMethod);
	    
	    if (!aspect_isMsgForwardIMP(targetMethodIMP)) {
	        // Make a method alias for the existing method implementation, it not already copied.
	        const char *typeEncoding = method_getTypeEncoding(targetMethod);
	        SEL aliasSelector = aspect_aliasForSelector(selector);
	        if (![klass instancesRespondToSelector:aliasSelector]) {
	            __unused BOOL addedAlias = class_addMethod(klass, aliasSelector, method_getImplementation(targetMethod), typeEncoding);
	            NSCAssert(addedAlias, @"Original implementation for %@ is already copied to %@ on %@", NSStringFromSelector(selector), NSStringFromSelector(aliasSelector), klass);
	        }
	
	        // We use forwardInvocation to hook in.
	        class_replaceMethod(klass, selector, aspect_getMsgForwardIMP(self, selector), typeEncoding);
	        AspectLog(@"Aspects: Installed hook for -[%@ %@].", klass, NSStringFromSelector(selector));
	    }
}
	```

	* aspect_hookClass 做了哪些事情？

		```
			Class statedClass = self.class;
			Class baseClass = object_getClass(self);
			NSString *className = NSStringFromClass(baseClass);
		```
		
		self.class 實際上調用的是
		
		```
			+ (Class)class {
	    		return self;
			}
		```
		
		返回的是類對象本身，
		
		而 object_getClass(self) 拿到的實際上是對象的 isa 指針
		
		```
			Class object_getClass(id obj)
			{
			    if (obj) return obj->getIsa();
			    else return Nil;
			}
	
		```
		
		```
			// Already subclassed
			if ([className hasSuffix:AspectsSubclassSuffix]) {
				return baseClass;
		
		        // We swizzle a class object, not a single object.
			}else if (class_isMetaClass(baseClass)) {
		        return aspect_swizzleClassInPlace((Class)self);
		        // Probably a KVO'ed class. Swizzle in place. Also swizzle meta classes in place.
		    }else if (statedClass != baseClass) {
		        return aspect_swizzleClassInPlace(baseClass);
		    }
		```
		
		如果 isa 指針指向的類是有 Aspects 後綴的，表示已經被 hook 過了，就直接返回這個類。
		
		如果類沒有前綴，且是元類，則調用 aspect_swizzleClassInPlace。
		
		如果非元類也非元類，同時類對象與 isa 指針指向的類非同一個，表示是 KVO 產生的類，因為 KVO 會在 runtime 產生一個中間類達成 KVO 機制，這時候對 KVO 產生的中間類調用 aspect_swizzleClassInPlace。
		
		```
			static Class aspect_swizzleClassInPlace(Class klass) {
		    NSCParameterAssert(klass);
		    NSString *className = NSStringFromClass(klass);
		
		    _aspect_modifySwizzledClasses(^(NSMutableSet *swizzledClasses) {
		        if (![swizzledClasses containsObject:className]) {
		            aspect_swizzleForwardInvocation(klass);
		            [swizzledClasses addObject:className];
		        }
		    });
		    return klass;
		}
		```
		_aspect_modifySwizzledClasses 會傳入一個 NSMutableSet 紀錄方法交換的類名，由於返回的 set 是全局的，且做了線程安全保護，如果發現當前的類名不在這個 set 裡面，則調用 aspect_swizzleForwardInvocation 這個方法，並且把當前類名加入 set 裡面。
		
		如果上述都不滿足，則 runtime 動態產生帶有 Aspects 後綴的子類。
		
		```
		// Default case. Create dynamic subclass.
		const char *subclassName = [className stringByAppendingString:AspectsSubclassSuffix].UTF8String;
		Class subclass = objc_getClass(subclassName);
	
		if (subclass == nil) {
			subclass = objc_allocateClassPair(baseClass, subclassName, 0);
			if (subclass == nil) {
	            NSString *errrorDesc = [NSString stringWithFormat:@"objc_allocateClassPair failed to allocate class %s.", subclassName];
	            AspectError(AspectErrorFailedToAllocateClassPair, errrorDesc);
	            return nil;
	        }
	
			aspect_swizzleForwardInvocation(subclass);
			aspect_hookedGetClass(subclass, statedClass);
			aspect_hookedGetClass(object_getClass(subclass), statedClass);
			objc_registerClassPair(subclass);
		}
	
		object_setClass(self, subclass);
		```
		
		這邊大量用了 runtime，
		
		1. objc_getClass 先取得 Aspects 新創建的類
		2. 如果找不到，則用 objc_allocateClassPair 創建一個
		3. aspect_swizzleForwardInvocation 這個方法做比較多事情，稍後一起說，主要是交換了方法的具體實現
		4. 	aspect_hookedGetClass(subclass, statedClass);
			aspect_hookedGetClass(object_getClass(subclass), statedClass);
			這兩個把剛剛新創的類 isa 指針以及 class 都指向了 statedClass，也就是這個類對象本身
		5. 最後 objc_registerClassPair(subclass) 註冊新創的類
		6. object_setClass(self, subclass) 把調用的對象 isa 指向新創的類。也就是 self -> subClass -> statedClass。然後返回新創的類。
	
		到目前為止，就完成了 aspect_hookClass，並把 self hook 成了 xxx_Aspects_。

	* aspect_swizzleForwardInvocation 做了哪些事？

		```
		static void aspect_swizzleForwardInvocation(Class klass) {
	    NSCParameterAssert(klass);
	    
	    // If there is no method, replace will act like class_addMethod.
	    IMP originalImplementation = class_replaceMethod(klass, @selector(forwardInvocation:), (IMP)__ASPECTS_ARE_BEING_CALLED__, "v@:@");
	    
	    if (originalImplementation) {
	        class_addMethod(klass, NSSelectorFromString(AspectsForwardInvocationSelectorName), originalImplementation, "v@:@");
	    }
	    AspectLog(@"Aspects: %@ is now aspect aware.", NSStringFromClass(klass));
	}
		```
		
		這裏把 forwardInvocation 替換成了 Aspects 定義的 __ASPECTS_ARE_BEING_CALLED__ 實現，如果成功返回，則在類上添加 __aspects_forwardInvocation 名的方法，具體實作就是 __ASPECTS_ARE_BEING_CALLED__。
		
	* __ASPECTS_ARE_BEING_CALLED__ 做了事情很多，但也是精華所在：
		1. 把從 hook forwardInvocation 拿到的 invocation 中拿到原始 selector 並且轉換成 aspects alias 的 selector，然後設置給 invocation
		2. 利用關聯對象拿到實例對象的 AspectsContainer 以及類對象的 AspectsContainer
		3. 初始化 AspectInfo，傳入 self 及 invocation
		4. 接著先執行定義成先調用的類及實例對象的 block

			```
			// Before hooks.
		    aspect_invoke(classContainer.beforeAspects, info);
		    aspect_invoke(objectContainer.beforeAspects, info);
			```
		5. 直接取代

			```
			// Instead hooks.
		    BOOL respondsToAlias = YES;
		    if (objectContainer.insteadAspects.count || classContainer.insteadAspects.count) {
		        aspect_invoke(classContainer.insteadAspects, info);
		        aspect_invoke(objectContainer.insteadAspects, info);
		    }else {
		        Class klass = object_getClass(invocation.target);
		        do {
		            if ((respondsToAlias = [klass instancesRespondToSelector:aliasSelector])) {
		                [invocation invoke];
		                break;
		            }
		        }while (!respondsToAlias && (klass = class_getSuperclass(klass)));
		    }
			```
		6. 後調用

			```
			// After hooks.
		    aspect_invoke(classContainer.afterAspects, info);
		    aspect_invoke(objectContainer.afterAspects, info);
			```
			
			都是把第三步產生的 AspectInfo 傳入。
			
			這裡利用宏定義了調用方法 aspect_invoke，節省了方法調用出入棧的開銷成本。
			
			裡面做了兩件事情，依序調用每個 aspect，也就是你所 hook 的方法，並把 AspectInfo 上下文傳入你的 block，然後把調用過的 aspect 加入等待被移除的 array。
			
			```
			// This is a macro so we get a cleaner stack trace.
	#define aspect_invoke(aspects, info) \
	for (AspectIdentifier *aspect in aspects) {\
	    [aspect invokeWithInfo:info];\
	    if (aspect.options & AspectOptionAutomaticRemoval) { \
	        aspectsToRemove = [aspectsToRemove?:@[] arrayByAddingObject:aspect]; \
	    } \
	}
			```
			
			invokeWithInfo 這個方法挺有意思，它把原本的 invocation 參數加入，之前產生的 block 的方法簽名 invocation 中，然後調用你的 block。
			
		7. 如果直接取代那一段沒有正常執行，則呼叫原本方法的實現，並替換為原本的 selector
		8. 最後，前調用、直接取代以及後調用都做完了之後，把這三個階段完成的 aspect 方法移除
	* aspect_remove：
	
		```
		static BOOL aspect_remove(AspectIdentifier *aspect, NSError **error) {
	    NSCAssert([aspect isKindOfClass:AspectIdentifier.class], @"Must have correct type.");
	
	    __block BOOL success = NO;
	    aspect_performLocked(^{
	        id self = aspect.object; // strongify
	        if (self) {
	            AspectsContainer *aspectContainer = aspect_getContainerForObject(self, aspect.selector);
	            success = [aspectContainer removeAspect:aspect];
	
	            aspect_cleanupHookedClassAndSelector(self, aspect.selector);
	            // destroy token
	            aspect.object = nil;
	            aspect.block = nil;
	            aspect.selector = NULL;
	        }else {
	            NSString *errrorDesc = [NSString stringWithFormat:@"Unable to deregister hook. Object already deallocated: %@", aspect];
	            AspectError(AspectErrorRemoveObjectAlreadyDeallocated, errrorDesc);
	        }
	    });
	    return success;
	}
		```
		
		remove 一樣用了一個自旋鎖保證線程安全，
	
		先利用關聯對象拿到 AspectsContainer，移除 aspect（AspectIdentifier）以及 AspectsContainer 的關係。
		
		aspect_cleanupHookedClassAndSelector 清除之前 hook 的類以及 selector。
		
		```
		aspect.object = nil;
      	aspect.block = nil;
      	aspect.selector = NULL;
      	```
      	
      	清除 aspect 相關信息。
      	
  	* aspect_cleanupHookedClassAndSelector 這個方法也是精華所在
  		
  		```
  		Class klass = object_getClass(self);
	   	BOOL isMetaClass = class_isMetaClass(klass);
	    if (isMetaClass) {
	        klass = (Class)self;
	    }
  		```
  		
  		先拿到類的 isa 指針，如果指向的就是元類，則用自己，如果還不是元類，則用 isa 指針指向的類。
  		
  		然後拿到的類的方法實現位置，把之前交換的 aspect alias 後的 selector 與原本的 selector 交換回來。
  		
  		```
  		// Check if the method is marked as forwarded and undo that.
	    Method targetMethod = class_getInstanceMethod(klass, selector);
	    IMP targetMethodIMP = method_getImplementation(targetMethod);
	    if (aspect_isMsgForwardIMP(targetMethodIMP)) {
	        // Restore the original method implementation.
	        const char *typeEncoding = method_getTypeEncoding(targetMethod);
	        SEL aliasSelector = aspect_aliasForSelector(selector);
	        Method originalMethod = class_getInstanceMethod(klass, aliasSelector);
	        IMP originalIMP = method_getImplementation(originalMethod);
	        NSCAssert(originalMethod, @"Original implementation for %@ not found %@ on %@", NSStringFromSelector(selector), NSStringFromSelector(aliasSelector), klass);
	
	        class_replaceMethod(klass, selector, originalIMP, typeEncoding);
	        AspectLog(@"Aspects: Removed hook for -[%@ %@].", klass, NSStringFromSelector(selector));
	    }
  		```
  		
  		這裡是以類為基礎，所以假設你有同個類的兩個實例對象 hook 了同一個方法，先 remove 的會把另一個實例對象的 hook 方法也一併移除。
  		
  		```
  		// Deregister global tracked selector
    aspect_deregisterTrackedSelector(self, selector);
  		```
  		
  		aspect_deregisterTrackedSelector 接著把之前提到的全局字典紀錄每個類以及對應的 AspectTracker 對象取出，並移除 selector name，selector name 底下的 sub class tracker，並從字典中移除紀錄，ㄧ樣用 for-loop 遍歷直到父類指向 NSObject，則結束。
  		
  		```
	  		// Get the aspect container and check if there are any hooks remaining. Clean up if there are not.
	    AspectsContainer *container = aspect_getContainerForObject(self, selector);
	    
	    if (!container.hasAspects) {
	        // Destroy the container
	        aspect_destroyContainerForObject(self, selector);
	
	        // Figure out how the class was modified to undo the changes.
	        NSString *className = NSStringFromClass(klass);
	        if ([className hasSuffix:AspectsSubclassSuffix]) {
	            Class originalClass = NSClassFromString([className stringByReplacingOccurrencesOfString:AspectsSubclassSuffix withString:@""]);
	            NSCAssert(originalClass != nil, @"Original class must exist");
	            object_setClass(self, originalClass);
	            AspectLog(@"Aspects: %@ has been restored.", NSStringFromClass(originalClass));
	
	            // We can only dispose the class pair if we can ensure that no instances exist using our subclass.
	            // Since we don't globally track this, we can't ensure this - but there's also not much overhead in keeping it around.
	            //objc_disposeClassPair(object.class);
	        }else {
	            // Class is most likely swizzled in place. Undo that.
	            if (isMetaClass) {
	                aspect_undoSwizzleClassInPlace((Class)self);
	            }else if (self.class != klass) {
	            	aspect_undoSwizzleClassInPlace(klass);
	            }
	        }
	    }
  		```
  		
  		aspect_destroyContainerForObject 清除了關聯對象中的 AspectsContainer。
  		
  		如果類名包含了 aspect 後綴，則把後綴去除，然後把 self 的指針指向原本的類，如果不是直接調用 aspect_undoSwizzleClassInPlace。
  		
	* aspect_undoSwizzleClassInPlace：

		```
		static void aspect_undoSwizzleClassInPlace(Class klass) {
	    NSCParameterAssert(klass);
	    NSString *className = NSStringFromClass(klass);
	
	    _aspect_modifySwizzledClasses(^(NSMutableSet *swizzledClasses) {
	        if ([swizzledClasses containsObject:className]) {
	            aspect_undoSwizzleForwardInvocation(klass);
	            [swizzledClasses removeObject:className];
	        }
	    });
		}
		```
		
		_aspect_modifySwizzledClasses 也是之前提到的全局紀錄所有交換過的類，這邊還原了所有類的 forward invocation，並把類從全局 set 裡移除。
		
		```
		static void aspect_undoSwizzleForwardInvocation(Class klass) {
	    NSCParameterAssert(klass);
	    Method originalMethod = class_getInstanceMethod(klass, NSSelectorFromString(AspectsForwardInvocationSelectorName));
	    Method objectMethod = class_getInstanceMethod(NSObject.class, @selector(forwardInvocation:));
	    // There is no class_removeMethod, so the best we can do is to retore the original implementation, or use a dummy.
	    IMP originalImplementation = method_getImplementation(originalMethod ?: objectMethod);
	    class_replaceMethod(klass, @selector(forwardInvocation:), originalImplementation, "v@:@");
	
	    AspectLog(@"Aspects: %@ has been restored.", NSStringFromClass(klass));
	}
		```
		
		這裡做的就是，把之前交換過的 __aspects_forwardInvocation: 以及原本類的 forwardInvocation: 方法交換回來。
		
	如此一來就完成了所有的清除以及 Swizzling 回復作業了。
	
	這裏我們了解了：
	
	1. 怎麼產生新的 hook 類，並且與原本的類交換 invocation
	2. __ASPECTS_ARE_BEING_CALLED__ 怎麼調用你的 block
	3. 最後如何清除以及 swizzling 還原

	因為我們是從 aspect_hookClass 進入，沒想到裡面就做了這麼多事情，最後返回了一個 apsects 新建的類，我們需要拿這個產生的類做交換。
	
	所以讓我們再回來最一開始的 aspect_prepareClassAndHookSelector 方法。
	
4. aspect_prepareClassAndHookSelector 續：

	```
	Class klass = aspect_hookClass(self, error);
    Method targetMethod = class_getInstanceMethod(klass, selector);
    IMP targetMethodIMP = method_getImplementation(targetMethod);
	```
	
	這裡返回的 klass 就是 aspect 產生的類，由於它的 isa 指向的是原本的類，所以我們可以拿到原本類 selector 方法的具體實現位置。
	
	```
	if (!aspect_isMsgForwardIMP(targetMethodIMP)) {
        // Make a method alias for the existing method implementation, it not already copied.
        const char *typeEncoding = method_getTypeEncoding(targetMethod);
        SEL aliasSelector = aspect_aliasForSelector(selector);
        if (![klass instancesRespondToSelector:aliasSelector]) {
            __unused BOOL addedAlias = class_addMethod(klass, aliasSelector, method_getImplementation(targetMethod), typeEncoding);
            NSCAssert(addedAlias, @"Original implementation for %@ is already copied to %@ on %@", NSStringFromSelector(selector), NSStringFromSelector(aliasSelector), klass);
        }

        // We use forwardInvocation to hook in.
        class_replaceMethod(klass, selector, aspect_getMsgForwardIMP(self, selector), typeEncoding);
        AspectLog(@"Aspects: Installed hook for -[%@ %@].", klass, NSStringFromSelector(selector));
    }
	```
	
	如果方法實現不是走 `_objc_msgForward` 或 `_objc_msgForward_stret` 消息轉發，先把原本 selector 轉為 aspect alias selector，然後加到 aspects 新產生的類的方法列表中，接者，替換這個 selector 的方法實現為走 `_objc_msgForward` 或 `_objc_msgForward_stret` 的消息轉發機制。
	
## 總結

最後簡單做個總結吧，Aspects 整體思路：當你調用某個類或是某個實例對象的 hook 方法，並傳入你想 hook 的 selector，以及你要執行的 block，還有調用時機點。Aspects 會在內部產生新的 xx_aspects_ 類對象，修改 isa 指針，把你的 block 轉為自定義的 block AspectsInfo 結構體，裡面當然包含了原先的 block 以及轉換後的 block 簽名。接著，修改消息轉發流程，把 forwardInvocation hook 到 aspects_forwardInvocation，並在裡面根據具體調用時機點，調用 block 方法簽名，然後把原本類的 selector 實現改為走消息轉發方式，當類原本方法被調用的同時，實際上會經由消息轉發走到 aspects_forwardInvocation 並且調用 block。當調用完你的 block 後，會把 block 消息並且回復消息轉發機制，調用原本類的方法。

Aspects 源碼也不到一千行，但裡面運用到 AOP 以及 runtime 的許多技術，十分值得我們研究一番。

p.s. 這篇文章拖了好久ＲＲＲ