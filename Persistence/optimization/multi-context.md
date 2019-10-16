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
![passed in one coordinator](https://github.com/coolboy-ccp/Optimization/blob/master/Persistence/optimization/images/oneCoordinatorPass.png)
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

## 注意点
### 冲突
Core Data 会在不同的栈层之间比较这些版 本标识。这样一来，当要存储托管上下文对象的数据时，Core Data 将其与原来的持久化数据 进行比较，如果两者版本标识不同，就意味着这个持久化数据发生了改变，这就叫保存冲突。冲突会发生在以下两个位置:
* 在快照和持久化存储的行缓存数据之间, 会在多个上下文连接到同一个持久化存储协调器的情况下发生
* 在持久化存储的行缓存数据和SQLite的数据之间, 会在多个持久化存储协调器连接到同一个持久化存储的情况下发生
### 合并策略
1. NSManagedObjectMergeError(默认)

不会解决冲突，抛出NSManagedObjectMergeError(NSError)异常，异常的userInfo中包含了冲突的详细信息

2. NSRollbackMergePolicy

引起冲突的那个对象上的更改就会被丢弃。那些没有导致冲突的对象会被正常保存。

3. NSMergeByPropertyStoreTrumpMergePolicy

冲突的更改会被一个属性接一个属性地合并到某个对象上。如果某个属性只是在持久化存储里被更改了但是内存里没有更改，那么会使用前者。如果正好相反，那么在内存里的更改会被持久化。但是如果某个属性在持久化存
储和内存里都被更改了，就预示着到底会发生什么:那就是位于持久化 存储里的更改将会成为最终结果。
   
4. NSMergeByPropertyObjectTrumpMergePolicy

如果某个属性在持久化存储和内存里都被更改了，那么位于内存里的更改会成为最终的结果。

一般情况下，用来和网络通信的后台上下文使用NSMergeByPropertyObjectTrumpMergePolicy，UI上下文使用NSMergeByPropertyStoreTrumpMergePolicy。这样，确保了“一切数据以服务器为准”

5. 自定义合并策略

继承NSMergePolicy, 重写resolve()，根据业务需求创建合并策略

### Query Generation(iOS 10+)
如果在一个上下文种执行多个连续的读操作 (比如填充惰值或是取数据) 的话，而另一个上下文 同时将更改保存到同一个数据库。这样一来在第一个上下文上执行的读操作就会存在潜在的数据不一致问题。在iOS之后，使用
```
try! moc.setQueryGenerationFrom(NSQueryGenerationToken.current)
```
这个特性允许一个上下文在使用数据库时可以无视其他上下文对其状态的更改， 就好像这些更改完全没有发生一样。当另一个上下文的修改被保存后，Generation中的数据会被更新。使用Query Generation会导致在某个上下文中保存数据后以及在另一个上下文中合并上下文已保存通知之前会出现一个时间间隙。在这个时间段里，第一个上下文所读取到的数据可能会和内存中已有的数据不一致。这在进行删除操作时尤其值得注意。
### delete object
在大多数多个上下文的设置方式中，删除操作是可能会引起崩溃的。如果在某个上下文中有个惰值，并且这个惰值所对应的对象在另一个上下文中被删除了，那么第一个上下文就无法填充这个惰值。如果你尝试去填充这个惰值，那就会导致一个运行时异常，因为这个惰值所引用的数据已经不存在了。解决方法：
1. context.shouldDeleteInaccessibleFaults(iOS 9.0+) = true，如果我们在某个上下文上设置了这个属性 (默认设置就是 它)，然后这个上下文尝试填充一个在持久化存储中已经不存在的惰值的话，那么这个惰值对象 就会被简单地标记为已删除。这种方法有两个缺点:
   * 被删除对象上所有的关系都会变成nil，所有的属性也都会根据其类型被设置为零或者 nil。不过只有当对象所有的属性都是可选 (Optional) 类型时，上述的操作才会发生。如果对象的某个或某些属性不是可选类型的话，在这个对象被删除之后，再去访问这些不 可选类型属性中的某一个就会导致运行时崩溃。
   * 一旦开启了shouldDeleteInaccessibleFaults(Default == YES)，Core Data 就再也无法在非可选属性上执行验证了
   2. 标记删除, 在对象实体上添加deletedMark。 删除时，并不从删除存储，而是标记删除。在iOS9.0+，可以使用批删除一次性过滤所有被标记的object，然后统一删除。在iOS9.0以下，需要寻找一个合适的时间进行删除。这种策略存在一个漏洞：无法触发关系删除规则。如果被标记的实体里存在关系，就需要手动制定删除策略。



