#  Coredata_multiContext

## 并发模型
![async model](https://github.com/coolboy-ccp/Optimization/blob/master/Persistence/optimization/images/asyncModel.jpg)
## merge
1. 注册NSManagedObjectContextDidSaveNotification
2. mergeChangesFromContextDidSaveNotification(_:) 
* sample demo
```
func sampleMerge() {
    // sourceContext
    // targetContext
    NotificationCenter.default.addObserver(forName: .NSManagedObjectContextDidSave, object: sourceContext, queue: .main) { (notification) in
        targetContext.viewContext.perform {
            self.persistentConstainer.viewContext.mergeChanges(fromContextDidSave: notification)
        }
    }
}
```
* 流程图
![merge](https://github.com/coolboy-ccp/Optimization/blob/master/Persistence/optimization/images/merge.png)

不同上下文中的操作 必须完全分离，上下文之间的数据交换只能通过 “上下文已保存” 通知进行，切记不要在不同上 下文之间随意调度。
## 对象传递
### 共享NSPresistentStoreCoordinator
调用 perfomr 方法将对象的 ID 传递到另一个上 下文的队列上，然后调用 object(with objectID:) 方法重新实例化这个对象
* sample demo
```
func sample(_ objectIDs: [NSManagedObjectID]) {
     mainContext.perform {
        let objects: NSManagedObject = objectIDs.map { mainContext.object(with:$0)}
     }
} 
```
![passed in one coordinator](https://github.com/coolboy-ccp/Optimization/blob/master/optimization/images/Persistence/optimization/images/oneCoordinatorPass.png)
### 独立NSPresistentStoreCoordinator
通过该NSManagedObjectID对象的 URIRepresentation 来重建对象的 ID。可以使用持久化存储协调器的 managedObjectID(forURIRepresentation:) 方法来实现这个操作
```

func finishedBackgroundOperation(_ objects: [NSManagedObject]) { let ids = objects.map { $0.objectID }
separatePSCContext.perform {
let results = ids.map {
(sourceID: NSManagedObjectID) -> NSManagedObject in
let uri = sourceID.uriRepresentation()
let psc = separatePSCContext.persistentStoreCoordinator!
let targetID = psc.managedObjectID(forURIRepresentation: uri)! return separatePSCContext.object(with: targetID)
}
// ... results 现在可以在主队列中使用了 }
}
```
![passed in coordinators](https://github.com/coolboy-ccp/Optimization/blob/master/Persistence/optimization/images/coordinatorsPass.png)

## 并发设置
### 共享NSPresistentStoreCoordinator
![one coordinator](https://github.com/coolboy-ccp/Optimization/blob/master/Persistence/optimization/images/oneCoordinator.png)
### 多个NSPresistentStoreCoordinator
![coordinators](https://github.com/coolboy-ccp/Optimization/blob/master/Persistence/optimization/images/coordinators.png)
### sub context
* 主线程上下文做为私有上下文的子上下文

![sub contest](https://github.com/coolboy-ccp/Optimization/blob/master/Persistence/optimization/images/subContext.png)
* 临时子上下文

![temp sub contest](https://github.com/coolboy-ccp/Optimization/blob/master/Persistence/optimization/images/tempSubContext.png)



