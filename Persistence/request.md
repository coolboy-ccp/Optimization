#  Coredata_Fetch
## NSFetchRequest获取流程

1. context请求NSPresistentStoreCoordinator
2. NSPresistentStoreCoordinator请求NSPresistentStore
3. NSPresistentStore将请求转换成SQL语句，并将SQL语句发送给SQLite
4. SQLite执行SQL语句，并将匹配结果返回给NSPresistentStore
5. NSPresistentStore实例化托管对象，将托管对象返回给NSPresistentStoreCoordinator
   如果是惰性对象，实例化对象时用对象ID(Core Data 保证在一个托管对象上下文里，无论你通过什么 方式，只会得到唯一一个表示某块数据的对象)创建的轻量对象。
6. NSPresistentStoreCoordinator将对象数组返回给context
7. If includesPendingChanges(Default is true) == true,
   在返回请求的结果之前，context会考虑正在等待进行的更改，并相应的更新结果。等待进行的更改是指context里已经修改(insert, update, delete), 但还未保存。
8. 调用者获取到托管对象数组

上述操作是同步操作，流程如下:
![request](https://github.com/coolboy-ccp/Optimization/blob/master/Persistence/request.png)

## Fault properties & 数据填充流程
If returnsObjectAsFaults(Defaults is true) == true, NSFetchRequest在返回结果时返回一个惰性值。惰性值是一些没有实际数据的轻量级对象，它们会在使用时被填充。
1. willAccessValue(forKey:)属性为惰性值
2. NSPresistentStoreCoordinator调用newValuesForObject(withobjectID:withcontext:)向NSPresistentStore请求与对象ID相关的数据
3. 如果缓存数据有效，NSPresistentStore会在缓存中检索数据
   缓存数据是否失效是由上下文的 stalenessInterval 属性来决定的。默认情况下，它被 设置为 0，这意味着缓存的数据永远不会失效，如果存在缓存，持久化储存协调器总会 返回缓存里的数据。如果你设置 stalenessInterval 属性为正值，那么只有当缓存数据 的最后更新时间小于失效的时间间隔 (单位是秒) 时缓存数据才会被使用。
4. 如果缓存数据失效，NSPresistentStore需要从SQLite中检索数据
5. 填充惰性对象数据

执行普通的获取请求后 (即其中的 returnsObjectsAsFaults 和 includesPropertyValues 属性 都是 true)，所有你请求的数据会被加载到行缓存里。这样的结果是，填充返回的惰值的开销是 相当廉价的 — 这是在较高内存占用和更快填充返回的惰值之间的一个权衡。
但是，通过设置 includesPropertyValues 属性为 false，你可以改变特定获取请求的默认行为， 防止它从数据库里加载除了对象 ID 之外的任何属性值。

惰性数据填充流程如下图:
![lazy object](https://github.com/coolboy-ccp/Optimization/blob/master/Persistence/FaultProperties.png)
## 刷新对象
当然，你也可以反过来把一个已经实体化的托管对象转成一个惰值。要做到这一点，可以为这 个对象调用上下文的 refresh(_ object:mergeChanges:) 方法。这个方法的第二个参数， mergeChanges，只在对象有未保存的更改的时才起作用。在这种情况下，传一个 true 值并不 会把对象变成惰值;相反，它会从行缓存里更新那些不变的属性，并保留所有未保存的更改。 这几乎总是你想要做的 (refreshAllObjects() 方法的做法也是如此)。
如果你指定 mergeChanges 为 false，这个对象会被强转成一个惰值，未保存的更改也会丢失。 所以使用它时需要非常谨慎，尤其要处理的关系上有未保存的更改时。在这种情况下，强行把 一个对象变成惰值可能会在你的数据里引入参照完整性问题 (referential integrity issue)。
获取请求有个叫 shouldRefreshRefetchedObjects 的选项，它会导致上下文自动刷新所有已 存在于上下文、同时也是获取结果一部分的对象。默认情况下，一个已存在的实体化对象不会 被获取请求所改变。设置 shouldRefreshRefetchedObjects 为 true 是一种确保返回的对象具 有持久化存储里最新值的便捷方法。
## Result Type
* managedObjectIDResultType
仅获取对象的 ID, 返回 NSManagedObjectID 实例的数组而不是通常的托管对象。但是请注意，这样的获取请求 仍然会从数据库中加载匹配行的所有数据，并更新协调器的行缓存。如果你想防止这种行为，那么你还是需要把获取请求的 includesPropertyValues 属性设为 false。
* countResultType
在概念上等同于使用 count(for request:) 而不是execute(_ request:) 方法。任何时候，如果你只需要知道结果的数量，请务必使用这个结果类 型，而不是执行一个常规的获取请求然后对其结果计数。
* dictionaryResultType
一个使用字典结果类型的获取请求将不再返回一个托管对象的数组来表示你请求的数据，而是返回一个包含原始数据的字典的数组。
* sample code
```
let request = NSFetchRequest<NSManagedObject>(entityName: "Employee")
request.resultType = .dictionaryResultType
let salaryExp = NSExpressionDescription() salaryExp.expressionResultType = .doubleAttributeType salaryExp.expression = NSExpression(forFunction: "average:",
arguments: [NSExpression(forKeyPath: "salary")]) salaryExp.name = "avgSalary"
request.propertiesToGroupBy = ["type"] request.propertiesToFetch = ["type", salaryExp]
_ = try! context.fetch(request)
```
propertiesToGroupBy, 表示在SELECT之前分组的属性名称；propertiesToFetch，指定那些属性被获取

## Batch Fetch
* sample code:
```
let request = NSFetchRequest<NSFetchResult>(entityName: "")
request.returnsObjectsAsFaults = false
request.fetchBatchSize = 20
```
### 步骤
1. 持久化存储将所有主键(对象ID)加载到内存中，并把它们转交回协调器。在这一步，谓 词和排序描述符仍然会被使用，但是这个查询的结果是只是一个对象 ID 的列表，而不 是与它们相关联的所有数据。
2. 持久化存储协调器创建一个特殊的，由这些对象ID组成的数组，并将这个数组返回给 上下文。请注意这个数组不是像我们上面看到的惰值的数组。因为数组背后的对象 ID 是一定的，所以数组的数目和里面的元素的位置也是固定的。不过这个数组没有被任何 数据填充 — 它在必要的时候才会去获取数据。
3. 这个分批处理的数组注意到它缺少你正尝试访问的元素的数据，它会要求上下文加载你 请求的索引附近数量为 fetchBatchSize 的一批对象。
4. 像往常一样，这个请求被持久化存储协调器转发到持久化存储，在那里执行适当的SQL 语句来从 SQLite 加载这批数据。原始数据被存储在行缓存里，托管对象则被返回给协 调器。
5. 因为我们已经设置了returnsObjectsAsFaults为false，协调器会要求存储提供全部数 据，然后用这些数据来填充对象，并将这些对象返回给上下文。
6. 这一批次的数组将返回你所请求的对象，并持有本批次里的其他对象，这样接下来如果 你需要使用其中某个对象的话，就不必再次获取了
当我们遍历数组时，Core Data 会额外加载几批所需数据之外的对象，并以最近使用作为原则 来保持少量批次，而较早的批次将会被释放。因此，使用 fetchBatchSize 允许我们通过一种内 存非常高效的方式来遍历超大的对象集合。
值得注意的是，将分批次获取和设置 returnsObjectsAsFaults 为 false 组合起来使用是有意义 的。这是因为我们一次只加载一小批的对象的数据。而且，我们很有可能是要立即使用这批数 据 — 比如，用于填充 table 或 collection view。
## Async Fetch
* sample code:
```
let fetchRequest = NSFetchRequest<Mood>(entityName: "Mood") let asyncRequest = NSAsynchronousFetchRequest(
fetchRequest: fetchRequest) { result in
if let result = result.􏰀nalResult { // 获取到结果了
} }
try! context.execute(asyncRequest)
```



