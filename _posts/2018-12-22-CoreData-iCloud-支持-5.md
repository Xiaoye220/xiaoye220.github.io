---
layout: post
title: iOS - CoreData iCloud 支持（5）
date: 2018-12-22
Author: Xiaoye 
tags: [CoreData]
excerpt_separator: <!--more-->
toc: true
---

在 iOS 10 的时候被 `deprecated`，现在打开[官方资料](https://developer.apple.com/library/archive/documentation/DataManagement/Conceptual/UsingCoreDataWithiCloudPG/Introduction/Introduction.html#//apple_ref/doc/uid/TP40013491-CH1-SW1)可以看到苹果报的提示，所以用的时候还是注意一些吧

![1.png](../images/2018-12-22-CoreData-iCloud-支持-5/1.png)



<!--more-->


### iCloud 支持
1. 在 Capabilities 中开启 iCloud 支持

2. 持久化存储协调器  （NSPersistentStoreCoordinator） 添加 持久化存储 （ NSPersistentStore）   时添加一下参数

```swift
// "iCloud.com.xx.xxxxx" 是在开启 iCloud 支持时给的 identifier
let options = [NSPersistentStoreUbiquitousContentNameKey: "DontStarve",
               NSPersistentStoreUbiquitousContainerIdentifierKey: "iCloud.com.xx.xxxxx"]
do{
    try coordinator.addPersistentStore(ofType: NSSQLiteStoreType, configurationName: nil, at: url, options: options)
} catch {
    let nserror = error as NSError
    fatalError("Unresolved error \(nserror), \(nserror.userInfo)")
}
```

3. 添加监听

```swift
func registerForiCloudNotifications() {
    let notificationCenter = NotificationCenter.default
    notificationCenter.addObserver(UserCoreDataStack.shared, selector: #selector(storesWillChange(_:)), name: NSNotification.Name.NSPersistentStoreCoordinatorStoresWillChange, object: persistentStoreCoordinator)
    
    notificationCenter.addObserver(UserCoreDataStack.shared, selector: #selector(storesDidChange(_:)), name: NSNotification.Name.NSPersistentStoreCoordinatorStoresDidChange, object: persistentStoreCoordinator)
    
    notificationCenter.addObserver(UserCoreDataStack.shared, selector: #selector(persistentStoreDidImportUbiquitousContentChanges(_:)), name: NSNotification.Name.NSPersistentStoreDidImportUbiquitousContentChanges, object: persistentStoreCoordinator)
    
}

/// 同步一条数据完成
@objc func persistentStoreDidImportUbiquitousContentChanges(_ notification: Notification) {
    self.managedObjectContext.perform { () -> Void in
        self.managedObjectContext.mergeChanges(fromContextDidSave: notification)
    }
}

/// storesWillChange
@objc func storesWillChange(_ notification: Notification) {
    self.managedObjectContext.perform { () -> Void in
        if self.managedObjectContext.hasChanges {
            self.managedObjectContext.saveOrRollback()
        }
        self.managedObjectContext.reset()
    }
}

/// storesDidChange
@objc func storesDidChange(_ notification: Notification) {
    self.managedObjectContext.perform { () -> Void in
        if self.managedObjectContext.hasChanges {
            self.managedObjectContext.saveOrRollback()
        }
    }
}
```

接下来使用 CoreData 会自动和 iCloud 同步