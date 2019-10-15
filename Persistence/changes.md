#  Coredata_Changes
## Changes tracking
* 两次save()之间, context.save()会触发一个NSManagedObjectContextDidSave 通知，这个通知包含自从上次 保存以来所有更改的信息。
* 两次processPendingChanges()之间, 这个方法通常会在两次保存之间被多次调用 — 比如它会在每个传递给上下文的 perform 方法的闭包执行之后被调用。在调用 processPendingChanges 时会触发一个 .NSManagedObjectContextObjectsDidChange 通知，这个通知包含自从上次发出这 类通知以来所有更改的信息。
### objects-did-change
通常不需要自己调用 processPendingChanges 方法。Core Data 会自动地调用这个方法 (比 如在更改被保存时以及 perform 方法调用结束之前)。当它被调用的时候，会发生如下的事情:
1. 删除对象时，如果propagatesDeletesAtEndOfEvent (Default is true) == true, 则会根据删除规则跨关系对删除进行传递。如果propagatesDeletesAtEndOfEvent == false，删除在保存时才会传递
2. 关系里的删除和插入会传递给它对应的反向关系
3. 被挂起的更改会被合并，而且会被注册为一个撤消操作(如果上下文设置了撤消管理器 (undo manager))。
4. 上下文发出一个.NSManagedObjectContextObjectsDidChange通知。 这个通知的 userInfo 字典将包含被插入，更新，删除以及自从上次
processPendingChanges() 方法被调用之后被刷新的对象集合。

在被更改的对象上调用 changedValuesForCurrentEvent 方法会返回一个精确 到包含属性粒度级别的更改信息条目的字典:这个字典的键是被更改的属性的名称，而字典的 值是这些属性的旧值。
### save之间的changes tracking
1. object changes tracking
   * hasChanges, 对象有更改，需要被保存
   * isInserted, 对象时新创建的，并且还没有被保存过
   * isDeleted, 对象被删除，在下次保存时会从数据库里删除
   * isUpdated, 对象已被更改

一旦你调用了某个 Core Data 属性的 setter 方法，无论你是不是真的修改了它的值， hasChanges 和 isUpdated 都会被修改为 true。因为这个属性会被 Core Data 标记为已更新， 所以这就能和我们后面会描述的冲突检测过程相关联上了。

如果你想了解一个托管对象的值和它对应的持久化数据相比是否实际发生了更改，那么你应该 检查它的 hasPersistentChangedValues。这个标志位使用该对象的当前值和它最后的持久化 状态进行比较，并且只在这些值确实不同的时候是 true。请注意，这个比较并不是和持久化存 储里的数据相比，而是和上次保存的数据或者是从存储里获取的数据进行比较。

调用托管对象的 changedValues 方法会返回一个字典，其中包含该对象自从上次被保存或被获取以来所更改的 键和新值。如果你想知道任意属性 (而不仅仅是那些已被更改的属性) 的旧值，你可以调用 committedValues(forKeys:) 方法。如果你要获得所有的旧值，直接给这个方法传一个 nil 即可。

2. context changes tracking
   * hansChanges, insertedObjects, updatedObjects, deletedObjects
   
对象可以包含在不止一个集合中。比方说，你先修改了一个对象，然后再删除它，这 个对象将会同时在 updatedObjects 和 deletedObjects 集合里被追踪到。

## 保存更改
### 调用save时，经历以下步骤
1. processPendingChanges方法被调用，并发出一个上面描述过的“对象已更改”通知。
2. NSManagedObjectContextWillSave通知
3. 对所有更改的对象进行验证。如果验证失败，保存过程会被中止，并抛出一个类型为 NSManagedObjectValidationError 或者 NSValidationMultipleErrorsError 的错误。
4. willSave方法在所有的有未保存更改的托管对象上被调用。

   你可以在你的 NSManagedObject 子类里覆盖这个方法，比如用它来对自定义数据类型 的序列化进行延迟处理等。如果这个时候你对托管对象做了进一步的更改 (或者在上一 步验证时进行了更改)，Core Data 将按照顺序循环地调用 processPendingChanges 方 法、然后再次验证，并在所有未保存的对象上调用 willSave，直到达到一个稳定的状态。
   你的责任是不要在这里创造一个死循环。举个例子，如果你要在 willSave 方法里做更 改，那么你应该在调用 setter 方法之前测试设置的值和已有值是否相等，因为就算要设 置的值和原来是相同的，对属性进行设置这个操作也会被当成一个更改。
5. 创建NSSaveChangesRequest并发送给持久化存储协调器。这个保存请求包含四组对 象:已插入的，已更新的，已删除的和已锁定的。
   已锁定的对象包括那些虽然没有被更改，但仍应参与冲突检测过程的对象。在此期间， 如果这些对象中的任意一个在持久化存储里发生了更改，那么保存将会失败。你可以通 过调用上下文的 detectCon􏰀icts(for object:) 方法来将未更改的对象加入到冲突检测中 去。
6. 持久化存储协调器通过调用自己的obtainPermanentIDs(forobjects:)方法来从存储里 获取新插入的对象的永久对象 ID。
   因为只有持久化存储拥有数据表中对象的主键的最终决定权，所以这一步是必需的。在 你插入一个新的对象时，上下文会使用一个临时 ID，这个 ID 在保存请求执行过程中被 会替换掉。(你可以使用 NSManagedObjectID 的 isTemporaryID 标志位来检查一个 ID 是否是临时的)
7. 持久化存储协调器将请求转发给持久化存储。
8. 持久化存储会检查从上下文最后一次获取后，你想保存的对象的数据在行缓存里是否被 更改过。
   Core Data 为每个托管对象维护所谓的原始数据快照。这些快照表示持久化存储里数据 的最后已知状态。在保存过程中，这些快照可以用来和行缓存里的数据相比:如果行缓 存里的数据已更改，保存会失败或者根据上下文的合并策略进行处理。我们之后将仔细 讨论冲突的情况，但是现在让我们假设不存在冲突并且保存可以正常进行。
9. 保存请求会被翻译成一个更新SQLite数据库里的数据的SQL查询语句。
   这是另一个可能发生冲突的地方。因为从上一次获取待保存对象上的数据之后，SQLite 数据库里的数据可能已经发生了更改。Core Data 如何处理这种情况也取决于上下文的 合并策略。现在让我们再次假设没有冲突发生。
10. 成功保存后，持久化存储的行缓存会被更新为新的值。didSave方法在所有被保存的托管对象上被调用。
11. 发送NSManagedObjectContextDidSave。

   这个通知的 userInfo 字典包含已被插入，更新和删除的对象集合。使用这个通知的一个 主要场景是:通过把通知作为参数传入并调用另一个托管对象上下文的 mergeChanges(fromContextDidSave:) 方法，来把已保存的更改合并到这个上下文。 
### 验证
你可以在 Xcode 的数据模型编辑器里设置简单的验证规则，比如整数属性的最小值和最大值。 当然你也可以通过代码来指定更复杂的规则。
验证在两个层级上工作:属性层级和对象层级。对于属性层级的验证，你可以在你的托管对象 子类里为每个属性实现单独的验证方法。这些方法必须遵循 validate<PropertyName> 这种命 名规则 — 比如:
```
public func validateLongitude(
_ value: AutoreleasingUnsafeMutablePointer<AnyObject?>) throws
{
guard let l = (value.pointee as? NSNumber)?.doubleValue else { return } if l < -180 || l > 180 {
throw propertyValidationError(forKey: "longitude", localizedDescription: "longitude has to be in range -180...180")
} }
```
使用propertyValidationError(forKey:localizedDescription:) 抛出异常
```
public func validateLongitude(
_ value: AutoreleasingUnsafeMutablePointer<AnyObject?>) throws
{
guard let l = (value.pointee as? NSNumber)?.doubleValue else { return } if abs(l) > 180.0 {
let shift = -l / abs(l) * 180.0
let truncated = l.truncatingRemainder(dividingBy: 180.0) value.pointee = NSNumber(value: shift + truncated)
} }
```
合理的修复无效值


除了属性层级的验证，你还可以实现操作跨属性的验证规则。为了实现这个目的，你需要覆盖 validateForInsert，validateForUpdate 或者 validateForDelete 方法。你应该在你的覆盖方法 的实现里调用 super，不然属性级别的验证不会发生。
### 冲突
Core Data 采用一种叫乐观锁 (optimistic locking) 的方法来处理冲突。这种方法之所以被称为乐观，是因为它把冲突的检查推迟到了上下文被保存的时候。
每个上下文都维护着每个托管对象数据的快照，这些快照表示的是 每个对象最近的已知的持久化状态。当你保存一个上下文时，这个快照会被用来和行缓存里的 数据以及在 SQLite 里的数据进行比较，以确保一切都没有更改。如果确实有更改，Core Data 会使用上下文的合并策略来解决冲突。如果你不指定合并策略，Core Data 的默认策略会直接 抛出一个 NSManagedObjectMergeError。这个错误将包含关于哪里出错的详细信息。

内置合并策略:
* NSRollbackMergePolicy, 直接丢弃引起冲突的对象的修改
* NSOverwriteMergePolicy, 忽略冲突并s持久化所有更改
* NSMergeByPropertyStoreTrumpMergePolicy, 发生冲突时保留存储里的数据
* NSMergeByPropertyObjectTrumpMergePolicy, 发生冲突时保留内存中的更改

## 批量更新
批量更新绕过了托管对象上下文和持久化存储协调器，它会直接对 SQLite 数 据库进行操作。如果你用这种方式更新一个属性或者删除一条记录，无论是托管对象上下文还 是协调器，都不会感知到这个变化。如果你不做一些额外的工作，这意味着所有常用的更新用 戶界面的机制 (比如 NSFetchedResultsController) 都将无法正常工作。另外，如果你没有考虑 到要手动进行更改的话，你可能会在后续保存时遇到冲突。


由于批量更新直接在 SQLite 级别上操作，所以如果行缓存包含了所有或是部分受影响的对象 的数据，那么在一次批量更新后行缓存里的数据也可能会过时。所以，仅通过调用上下文的 refresh(_:mergeChanges:) 甚至 refreshAllObjects() 方法来刷新托管对象是不够的 — 虽然它 们可以对象将变成惰值，但是行缓存仍然持有那些旧的数据。下次在你访问其中这些对象的某 个属性时，行缓存里过时的数据仍将会被再次使用。
### 数据更新方案
* 使用托管对象上下文类上的静态方法mergeChanges(fromRemoteContextSave:into:)。
   它的第一个参数接受一个字典，这个字典中应该包含 NSInsertedObjectsKey、 NSUpdatedObjectsKey 或 NSDeletedObjectsKey (取决于你所做的更改) 键，其对应 的值为包含对象 ID 的数组。在幕后，这个方法会为那些已经在上下文中注册过的对象 去从存储里获取新的数据，并更新行缓存。虽然这个 API 只在 iOS 9 和 macOS 10.11 之 后才可用，但是使用它是我们的首选方案。
* 使用获取请求重新获取数据，并刷新获取请求结果中在上下文里注册过那部分对象的数 据。
   在你使用这个方法的时候，比较合理的做法是在获取请求里将结果类型设置为 .managedObjectIDResultType，从而避免创建不必要的托管对象实例。这样一来就算 是你只请求了对象 ID，行缓存仍然会被更新。

