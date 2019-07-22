---
layout: post
title: iOS - 归档 NSSecureCoding
date: 2019-05-31
Author: Xiaoye
tags: [iOS]
excerpt_separator: <!--more-->
toc: true
---

在使用 `NSKeyedArchiver` 归档 `URLResponse` 时发现 `unchiver` 出来的始终是 `nil` ，因此对 iOS 归档进行了一番研究查找原因

## 1. NSCoding , NSSecureCoding

我们要使一个对象支持归档，那么必须实现协议 `NSCoding` ，在 `iOS 6.0` 之后出了 `NSSecureCoding` 用来替代 `NSCoding`。那么 `NSSecureCoding` 相比 `NSCoding` 都有哪些优势呢

我们看官方的 [Document](<https://developer.apple.com/documentation/foundation/nssecurecoding>)，有以下解释

> Historically, many classes decoded instances of themselves like this:
>
> ```
> if let object = decoder.decodeObjectForKey("myKey") as MyClass {
>     // ...succeeds...
> } else {
>     // ...fail...
> }
> ```
>
> This technique is potentially unsafe because by the time you can verify the class type, the object has already been constructed, and if this is part of a collection class, potentially inserted into an object graph.

大概的意思就是使用 `NSCoding` 是不安全的，因为在`decode` 的时候，`NSCoding` 不会关心 `decode` 出来的是什么，只会先构建出一个对象，再去验证类型是否符合。这就有可能给攻击者机会在这一步 `decode` 出一个恶意的类型的对象，造成危害。

而 `NSSecureCoding` 能有效的避免这个问题，因此官方 API 都逐渐替换成了 `NSSecureCoding`。当 `NSKeyedArchiver` 的 `requiresSecureCoding` 为 `true` 时，那么对于实现了 `NSSecureCoding` 的类使用以下代码 `decode` 返回的会是 nil

```swift
if let object = decoder.decodeObjectForKey("myKey") as MyClass {
    // ...succeeds...
}
```

要想正确的 `decode` 一个 `NSSecureCoding` 的类型必须使用以下方法，会先进行类型验证，再构建对象

```swift
let obj = decoder.decodeObject(of:MyClass.self, forKey: "myKey")
```

## 2. 归档 NSSecureCoding 类型

这里就用 `URLResponse` 为例，因为在 `iOS 12` 部分 API 被 `deprecated`，所以代码分 iOS 12 之前和之后两种

### 2.1 实例方法

#### Archiver

```swift
func achiver<T: NSSecureCoding>(_ obj: T) -> Data {
    if #available(iOS 12.0, *) { 	// iOS 12.0 之后
        let encoder = NSKeyedArchiver(requiringSecureCoding: true)
        encoder.encode(obj, forKey: "obj")
        encoder.finishEncoding()
        return encoder.encodedData
        
    } else { 	// iOS 12.0 之前
        let data = NSMutableData()
        let encoder = NSKeyedArchiver(forWritingWith: data)
        encoder.requiresSecureCoding = true
        encoder.encode(obj, forKey: "obj")
        encoder.finishEncoding()
        return data as Data
    }
}
```

#### UnArchiver

```swift
func unarchiver<T: NSSecureCoding & NSObject>(_ data: Data) -> T? {
    if #available(iOS 12.0, *) { 	// iOS 12.0 之后
        do {
            let decoder = try NSKeyedUnarchiver(forReadingFrom: data)
            decoder.requiresSecureCoding = true
            let obj = decoder.decodeObject(of: T.self, forKey: "obj")
            decoder.finishDecoding()
            return obj
        } catch {
            print(error)
            return nil
        }
    } else {	// iOS 12.0 之前
        let decoder = NSKeyedUnarchiver(forReadingWith: data)
        decoder.requiresSecureCoding = true
        let obj = decoder.decodeObject(of: T.self, forKey: "obj")
        decoder.finishDecoding()

        return obj
    }
}
```

#### 使用

```swift
let obj = URLResponse()
// archiver
let achiverData = self.achiver(obj)
// unarchiver
let unchiverObj: URLResponse? = self.unarchiver(achiverData)
```

### 2.2 静态方法

#### Archiver

```swift
func achiver<T: NSSecureCoding>(_ obj: T) -> Data {
    if #available(iOS 12.0, *) {
        do {
            let data = try NSKeyedArchiver.archivedData(withRootObject: obj, requiringSecureCoding: true)
            return data
        } catch {
            print(error)
        }
        
    } else {
        let data = NSKeyedArchiver.archivedData(withRootObject: obj)
        return data
    }
}
```

#### UnArchiver

```swift
func unarchiver<T: NSSecureCoding & NSObject>(_ data: Data) -> T? {
    if #available(iOS 12.0, *) {
        do {
            let obj = try NSKeyedUnarchiver.unarchivedObject(ofClass: T.self, from: data)
            return obj
        } catch {
            print(error)
            return nil
        }
        
    } else {
        let obj = NSKeyedUnarchiver.unarchiveObject(with: data) as? T
        return obj
    }
}
```

##### 

## 3.归档自定义对象

自定义对象必须实现协议 `NSSecureCoding`

```swift
class Home: NSObject, NSSecureCoding {
    var address: String?
    
    static var supportsSecureCoding: Bool {
        return true
    }
    
    func encode(with aCoder: NSCoder) {
        aCoder.encode(self.address, forKey: "address")
    }
    
    required init?(coder aDecoder: NSCoder) {
        self.address = aDecoder.decodeObject(forKey: "address") as? String
    }
    
    init(address:String) {
        self.address = address
    }
}
```

```swift
class Person: NSObject, NSSecureCoding {
    
    var name: String?
    var age: Int = 0
    var home: Home?

    static var supportsSecureCoding: Bool {
        return true
    }
    
    func encode(with aCoder: NSCoder) {
        aCoder.encode(self.name, forKey: "name")
        aCoder.encode(self.age, forKey: "age")
        aCoder.encode(self.home, forKey: "home")
    }
    
    required init?(coder aDecoder: NSCoder) {
        // 备注(1)
        self.name = aDecoder.decodeObject(forKey: "name") as? String
		self.age = aDecoder.decodeInteger(forKey: "age")
        // 备注(2)
        self.home = aDecoder.decodeObject(of: Home.self, forKey: "home")
    }
    
    override init() {
        
    }
}
```

上面声明了两个自定义类型 `Person` 和 `Home` ，都实现了协议 `NSSecureCoding`，上面代码有两处备注需要注意的

* `备注(1)` ：对于基础类型 `String` 、`Int` 等因为没有实现协议 `NSSecureCoding` 因此无需使用方法 `decodeObject(of:forKey:)` 来 decode。但是对于 `Int` 、`Bool` 等类型有专门的方法进行转换，即以上 `decodeInteger(forKey:)` ，这里如果还是使用 `decodeObject(forKey:)` 会返回 nil

  这个还有一种方式避免这种问题，通过以下实现来统一使用，因为 `NSNumber`、`NSString` 都实现了协议 `NSSecureCoding` ，可以通过转换为  `NSNumber`、`NSString` 进行 encode 以及 decode

  ```
  func encode(with aCoder: NSCoder) {
      aCoder.encode(self.name as NSString?, forKey: "name")
      aCoder.encode(NSNumber(value: self.age), forKey: "age")
  
      aCoder.encode(self.home, forKey: "home")
  }
  
  required init?(coder aDecoder: NSCoder) {
      self.name = aDecoder.decodeObject(of: NSString.self, forKey: "name") as String?
      self.age = aDecoder.decodeObject(of: NSNumber.self, forKey: "age") as! Int
  
      self.home = aDecoder.decodeObject(of: Home.self, forKey: "home")
  }
  ```

* `备注(2)` ：因为 `Home` 类型实现了 `NSSecureCoding`，所以必须使用 `decodeObject(of:forKey:)` 方法 decode



对 `Person` 类型进行归档

```swift
let person = Person()
person.name = "Tommy"
person.age = 18
person.home = Home(address: "Fu zhou")

// archiver
let codedData = self.achiver(person)
// unarchiver
let encodedPerson: Person? = self.unarchiver(achiverData)

print(encodedPerson)
print(encodedPerson?.age)
print(encodedPerson?.name)
print(encodedPerson?.home?.address)
```

