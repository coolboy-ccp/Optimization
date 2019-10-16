#  Coredata_性能
## 层级
![optimization](https://github.com/coolboy-ccp/Optimization/blob/master/Persistence/optimization/images/optimization.png)
为了使用 Core Data，我们需要让我们的数据处于上下文层，也即由上下文和托管对象 所组成的层。我们需要下降到其他层才能把数据放到该层中来。因此为了优化性能，我们需要 尽可能地限制返回这些层的次数;这是几乎所有改善 Core Data 性能技术的关键部分。
## 避免获取请求

### 使用关系
![city-person](https://github.com/coolboy-ccp/Optimization/blob/master/Persistence/optimization/images/cityPerson.jpg)
* to-one
有一个city对象，需求是获取市长。有如下两种方案:
1. 访问city对象的mayor：
   如果 City 对象是一个惰值，类似我们 访问它的其他数据一样，数据需要先被填充。其次，一旦 City 对象实体化了，它的上下文会创建一个相应的 Person 对象作为这个市的市⻓，但是呢，这个 Person 对象也是一个惰值，除非它已经被访问过了。
   
   Core Data 强制对象的唯一性。如果 Person 对象已经在上下文 里注册过，那么 city 对象的 mayor 属性会指向这个实例。如果这个 Person 对象已经被完全实 体化了 (非惰值)，它的 mayor 属性将会指向这个被完全实体化的 Person 对象。
2. 在Person上执行一个request
   通过所有层并与文件系统交 互来检索对应的 Person 对象
* to-many
获取City对象里的residents
1. 访问city的residents属性
   如果 City 对象的属性已经被实体化了，residents 属性会变成 所谓的关系惰值 (relationship fault)。相应地，如果 City 对象是一个惰值，访问 residents 对 象会实体化它的属性。但是 residents 仍然会是一个关系惰值
   residents 属性返回的是关系惰值对象，它是一个 Set (或者 Objective-C 里的 NSSet)，在幕后， 它延迟了数据加载。只有当我们访问它的元素或者它的计数的时候 Core Data 才会解析关系并 且填充关系所指向的对象:Core Data 会把对应的 Person 对象插入到 Set 里。如果单个的 Person 对象没有在上下文里注册，它们会全是惰值。如果它们中的一些已经存在了 (无论是惰 值或者完全实体化)，那么这些存在的实例会被插入到集合里。
2. Person中执行reques

如果关系另一端的对象已经在上下文里注册过可能性较大，那么使用关系会更有意义，因为我 们不用再从 SQLite 中取回这些对象的数据。如果这些对象不太可能被注册过 (特别是大部分对 象都没注册过)，那么执行获取请求会更有意义。一个折中的做法:我们可以获取对多关系里的 对象，如果存在还是惰值的对象，我们为这些对象执行获取请求。

### 有序关系

有序关系在我们的模型中有时会很方便。但是，这同样代表了某些性能上的取舍。在碰到那些 需要插入或更新对象的场景时，一个有序的关系会更慢，这是因为 Core Data 还要管理和持久 化这些顺序。而对于无序的对多关系，Core Data 并不需要维护任何有关顺序的信息。如果我们需要按一定顺序取回对象，从有序的关系里取回预先排序好的数据会比先取回数据后再进行排 序要快。即使这个排序是在 SQLite 中完成的，也还是前者更快。

### 搜索特定的对象
检查上下文看是否存在满足我们正在查找的谓词的对象。如果存在，我们直接返回这个对象;否则的话我们执行获取请求。不过这种做法只在我们知道有且只有一个符合特定谓词的对象的情况下才有效。
```
extension NSManagedObject {
    static func registered(in context: NSManagedObjectContext, matching predicate: NSPredicate) -> Self? {
        for object in context.registeredObjects where !object.isFault {
            guard let result = object as? Self, predicate.evaluate(with: result) else { continue }
            return result
        }
        return nil
    }
    
    static func findOrFetch(in context: NSManagedObjectContext, matching predicate: NSPredicate) -> Self? {
        guard let object = registered(in: context, matching: predicate) else {
            let fetch = DBFetch<NSManagedObject>(context, "")
            return try? fetch.properties().first as? Self
        }
        return object
    }
}
```
在对象不是惰值的时候才计算谓词，否则会将惰值填充
### 类似单例的对象
通过userInfo实现primaryKey(iOS9之后使用constraints), 参考[MagicalRecord](https://github.com/magicalpanda/MagicalRecord)。这种实现在第一次调用的时候会执行请求，性能和之前一样(忽略userinfo的存储性能), 但在之后的调用就会特别快了。在对象被删除后，需要手动清除在userinfo字典里的缓存。

### 小数据集
另外的场景是处理相对小的数据集。要么事先你就知道整个 app 的对象数量不会超过几百个， 要么你知道特定的实体的对象数目不会超过几百个。
如果你的数据集很小，直接把整个数据库一次性加载到内存，并在内存里操作它们很可能是值
得的。比如你可以用一个获取请求把所有对象都加载到上下文里，然后存入一个数组，保证这
些对象会被强引用着。
### 优化获取请求
* 对象排序
SQLite 排序对象是很快的，特别是按建有索引的属性进行排序的话。导致获取后再进行排序时速度很慢的原因有许多。假如你使用任何Array 或 NSArray 的排序方法，这些方法会访问数组里的每个托管对象的属性，结果会导致数组里的每个对象都被实体化。
* 避免多个，连续的惰值
假设我们有多个是惰值的对象，如果
我们想访问它们的属性，会触发每个对象填充惰值，带来的开销会比较大。反而是批量实体化
所有对象会更快。我们可以执行一个获取请求来获取所有我们感兴趣的对象。
因为 Core Data 保证对象的唯一性。我们其实并不需要使用这个获取请求返回的结果，就能让 我们已经持有的对象更新。
请注意，这样带来的可能的性能提升很大程度取决于持久化存储协调器和 SQL 层的竞态程度。 如果我们知道对象的数据仍保持在行缓存中，而且持久化存储协调器没有被竞态访问，因为这 种情况下可以避免接触到 SQL 层，所以逐个填充惰值可能还是会更快一些。性能优化最好是仔 细考虑对比，之后再动手去做。

```
extension Collection where Iterator.Element: NSManagedObject { public func fetchFaults() {
guard !self.isEmpty else { return }
guard let context = self.first?.managedObjectContext
else { fatalError("Managed object must have context") }
let faults = self.􏰁lter { $0.isFault }
guard let object = faults.􏰁rst else { return }
let request = NSFetchRequest<Iterator.Element>() request.entity = object.entity request.returnsObjectsAsFaults = false
request.predicate = NSPredicate(format: "self in %@", faults) let _ = try! context.fetch(request)
} }
```
### 批量获取
获取请求返回的对象会被立刻呈现出来，所以我们希望这些对象的属性能立刻填充。如果没有
这样做的话，返回的对象会全是惰值:对象的数据只会被载入到行缓存里，只有当对象属性被
访问的时候才会被后续加载到上下文里。我们会为每个对象都付出和持久化存储协调器交互的
代价。
另外一个设置批次大小的重要的原因是:我们限制了数据从存储层传递到持久化存储协调器和 托管对象上下文的数据数量。这让结果相当的不同，特别是对于大的数据集而言;如果不设置 批次大小，Core Data 会把所有满足条件的对象移动到行缓存里，这样不仅占用内存还会花费 更多时间和消耗更多的电量。
如果我们设置了批次大小，我们会引入遍历数据集带来的新的开销:当用戶滚动列表的时候，
每隔一定的间隔，我们就不得不去获取新的下一批次的数据。不过通过设置合适的批次大小，
我们可以确保每批的开销相对较小，而且不会太频繁的去支付这个开销。
一个经验法则是，适合一屏显示项数的 1.3 倍会是一个合适初始值。需要记住的重要一点是，一屏显示所需的数量是设备相关的。对于 table view，我们可以直接用屏幕的高度除以行高，然后乘以 1.3 就很合适了.
### Fetched Results Controller
大部分 fetched results controller 的性能特质直接依赖于底层的获取请求。特别地，确保能够 利用 SQLite 存储里的索引来进行排序是非常重要的。下面我们会讨论更多有关索引的内容。
另外 fetched results controllers 可以使用持久化的缓存。这种缓存能加速后续使用的相同 fetched results controller。持久化缓存很适合那些在每次 app 启动之间很少改动的数据。使 用持久化缓存可以提高 app 的启动性能。同时它对于 fetched results controller 需要展示很大 数据集的时候也有帮助。
在创建 fetched results controller 的时候，你可以传入一个 cacheName 参数。这样就告诉 Core Data 使用持久化缓存。相应的获取请求的名称和 section 名字的 key path 需要在 app 里 是唯一的。NSFetchedResultsController 的文档里有更详细的介绍。
### 关系预加载
就像前面提到的，获取请求返回的对象默认没有填充它们的属性，这些数据只会加载到行缓存 里。只有设置 returnsObjectsAsFaults 为 false 的时候，对象才会被完整填充，也就是说，它 将不是一个惰值。对于关系而言，情况就复杂得多了。对于对一关系，获取请求会把目标对象 的 ID 作为获取的一部分取出。而对于对多的关系，则不会做额外的工作。
当我们访问关系的另一端的对象时，会触发一个惰值;对于对多的关系，这甚至会导致双重惰
值。
在我们知道确实需要访问关系的另一端的对象的情况下，我们可以让获取请求预加载这些对象:
request.relationshipKeyPathsForPrefetching = ["mayorOf"] 我们甚至可以通过使用 key path 来深入对象图的层级 (比如 “friends.posts” 会预加载
“friends” 关系指向的对象，然后对于每个对象，会预加载 “posts” 关系指向的对象)。
但是，需要着重指出的是，如果你需要通过关系的预加载来展示 table view，这通常是一种坏 代码的味道。如果类似的需求多次出现，你应该审视你的模型，问问自己对这些模型去规范化 (denormalizing) 会不会有所帮助。
### 索引&复合索引
索引能从两个不同的方向改善性能:一是可以显著提高排序的性能，二是可以显著提高使用谓
词的获取请求的性能。
必须要指出的是，添加索引也有代价:添加和更新数据时需要更新对应的索引，数据库文件也
会因为包含了索引而变得更大。

是否添加索引，取决于，在你的应用程序中，获取数据与修改数据频繁程度的比例。如果更新
或者插入非常频繁，最好不要添加索引。如果更新或插入不频繁，而查询和搜索非常频繁，那
么添加索引会是个好主意。

还有一个因素是数据集的大小:如果实体的数目相对较小，添加索引并不能给我们带来多少帮
助，因为数据库扫描所有数据也很快。但是如果数量巨大，添加索引就可能可以显著改善性能。
### Insert & Update
值得注意的是，在上下文层中插入新对象或者修改已有对象的操作开销都很小，因为这些操作 只接触到了托管对象上下文层，没有接触到 SQLite 层。
只有当这些改动被持久化到存储层的时候开销才会相对较高。因为持久化操作会涉及到协调器 层和 SQL 层。也就是说，调用 save() 是非常昂贵的。
性能优化的关键原理很简单，减少保存的次数就可以了。但是我们需要保持一个平衡:保存一 个大的改动会增加内存消耗，因为上下文会一直记录这些改动直到它们被保存;同时，与保存 小的改动相比，保存大的改动需要消耗更多的 CPU 资源和内存。不过一般而言，保存一个大的 改变集会比保存很多小的改动集代价要更低。
## 如何构建高效的数据模型
* 根据实际使用设计model
* 一对一关系通常可以被内联。假设我们的模型里有 Person 和 Pet 实体，它们之间的关系是一 对一，我们可以将这个信息内联到一个的实体里 - PersonWithPet - 它同时包含 personName 和 petName 两个属性。
* 去规范化，假设我们的模型里有雇员实体，雇员之间有一个关系来表 示谁管理谁。如果我们用 table view 来展示雇员信息，对于每个 cell，我们显示雇员的姓名和 向其汇报的其他雇员数量，我们可以使用最朴素的做法，通过查看对多关系来统计雇员数量。 但是这种做法开销会很大，因为对于每个雇员对象都需要填充关系惰值。
为了避免填充所有这些关系惰值，我们可以把向雇员汇报的组员数目放入到雇员对象中， Employee 实体有一个叫 teamMembers 的关系和一个叫 numberOfTeamMembers 的属性。 这是有效的数据冗余，用数据库的术语来说就是去规范化。虽然我们曾经被多次教导过，数据 冗余是不好的，但是在这种情况下，去规范化是能显著提高获取性能的一个有力工具。

SQLite 使用 WAL 机制来实现原子事务，WAL 机制的原理是:修改并不直接 写入到数据库而是写入到一个称为 WAL 的文件中;如果事务失败，WAL 中的记录会 被忽略，撤销修改;如果事务成功，它将在随后的某个时间被写回到数据库文件中，提 交修改。同步 WAL 文件和数据库文件的行为被称为检查点。


