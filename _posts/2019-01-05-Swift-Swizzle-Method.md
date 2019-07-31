---
layout: post
title: iOS - Swift Swizzle Method
date: 2019-01-05
Author: Xiaoye
tags: [Runtime]
excerpt_separator: <!--more-->
toc: true
---

本文环境
* Xcode 10
* Swift 4.2

在 Swift 中通过 `block` 替换方法会和 OC 中有些不同

以下代码在 num 为 0 时会崩溃，我们通过 swizzle method 对 0 进行处理，避免崩溃。注意以下方法我们添加了  `dynamic` 关键字，使其使用 runtime 机制

```swift
class Caculator: NSObject {
    @objc dynamic func caculator(_ num: Int) -> Int {
        print("caculator")
        return 1/num
    }
}
```

### 1. Block 替换方法

有时候，我们并不是很方便在原有类中添加一个新方法进行交换，那么我们可以通过一个 `block` 将一个方法的实现替换成 `block` 的实现。

```swift
extension Caculator {
    class func swizzleCaculatorWithBlock() {
        let originalSelector = #selector(Caculator.caculator)
        let originalMethod = class_getInstanceMethod(Caculator.self, originalSelector)!
        let originalImplementation =  method_getImplementation(originalMethod)
        
        typealias originalImp  = @convention(c) (Caculator, Selector, Int) -> Int
        
        // unsafeBitCast 将 originalImplementation 强制转换成 originalImp 类型
        // 两者的类型其实是相同的，都是一个 IMP 指针类型，即 id (*IMP)(id, SEL, ...)
        let originalClosure = unsafeBitCast(originalImplementation, to: originalImp.self)
        
        let newBlock: @convention(block) (Caculator, Int) -> Int = { obj, num  in
            print("swizzled with block")
            if num == 0 { return 0 }
            return originalClosure(obj, originalSelector, num)
        }
        
        // The signature of block should be method_return_type ^(id self, self, method_args …).
        let newImplementation = imp_implementationWithBlock(unsafeBitCast(newBlock, to: AnyObject.self))
        method_setImplementation(originalMethod, newImplementation)
    }
}
```

- @convention 关键字的解释可以见 [@convention](http://swift.gg/2016/05/18/swift-qa-2016-05-18/)
- originalImp 是一个 [IMP](https://developer.apple.com/documentation/objectivec/objective-c_runtime/imp?language=objc) 类型，即 `id (*IMP)(id, SEL, ...)` ，因此需要声明成 `(Caculator, Selector, Int) -> Int`
- `imp_implementationWithBlock` 中对 block 的说明是 `The signature of block should be method_return_type ^(id self, self, method_args …).` ，因此 `newBlock`  声明为 `(Caculator, Int) -> Int`



接着我们执行

```swift
Caculator.swizzleCaculatorWithBlock()
let result = Caculator().caculator(0) // result = 0，并且不会崩溃
```



### 2. 方法交换方法

首先我们创建一个需要交换的方法

```swift
extension Caculator {
    @objc func swizzledCaculator(_ num: Int) -> Int {
        print("swizzledCaculator")
        if num == 0 { return 0 }

        // 因为方法交换后，调用 swizzledCaculator 方法实际调用的是交换前 caculator 的方法
        return swizzledCaculator(num)
    }
}
```

接着进行方法交换

```swift
extension Caculator {
    class func swizzleCaculator() {
        let originalSelector = #selector(Caculator.caculator)
        let swizzledSelector = #selector(Caculator.swizzledCaculator)
        
        swizzleMethod(for: Caculator.self, originalSelector: originalSelector, swizzledSelector: swizzledSelector)
    }
    
    class func swizzleMethod(for aClass: AnyClass, originalSelector: Selector, swizzledSelector: Selector) {
        let originalMethod = class_getInstanceMethod(aClass, originalSelector)
        let swizzledMethod = class_getInstanceMethod(aClass, swizzledSelector)
        
        let didAddMethod = class_addMethod(aClass, originalSelector, method_getImplementation(swizzledMethod!), method_getTypeEncoding(swizzledMethod!))
        
        if didAddMethod {
            class_replaceMethod(aClass, swizzledSelector, method_getImplementation(originalMethod!), method_getTypeEncoding(originalMethod!))
        } else {
            method_exchangeImplementations(originalMethod!, swizzledMethod!)
        }
    }
}

```

同上，最后我们执行

```swift
Caculator.swizzleCaculator()
let result = Caculator().caculator(0) // result = 0，并且不会崩溃
```

