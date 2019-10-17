#  谓词

## 简单谓词
```
Object
let predicate = NSPredicate(format: "%K == value", #keyPath(Object.property))
predicate.evaluate(with: object)
```
* [官方文档](https://developer.apple.com/library/archive/documentation/Cocoa/Conceptual/Predicates/Articles/pCreating.html#//apple_ref/doc/uid/TP40001793-219639-BCIIHDCH)
* #keyPath, swift用来创建谓词K的语法，#keyPath修饰的属性必须是@objc的，其类必须继承NSObject

## 合并谓词
NSCompoundPredicate

## 常量谓词
```

protocol Hideable {
static var hiddenPredicate: NSPredicate { get }
}
extension Person: Hideable {
static var hiddenPredicate: NSPredicate {
return NSPredicate(format: "%K", #keyPath(Person.hidden)) }
}
extension City: Hideable {
static var hiddenPredicate: NSPredicate {
let predicate = NSPredicate(value: true)
return predicate }
}
```
Person可被隐藏，使用NSPredicate(value: true)可匹配所有的被隐藏的Person

## 匹配关系
* to-one
```
 let predicate = NSPredicate(format: "%K.%K > %lu", #keyPath(City.mayor), #keyPath(Person.age), 30)
```

匹配市长大于三十岁的城市

* to-many
```
let predicate = NSPredicate(format: "ANY %K.%K <= %lu", #keyPath(City.mayor), #keyPath(Person.age), 20)
```

只要一个城市的任意一个访客小于21岁，即可被匹配

## 子查询
```
let predicate = NSPredicate(
format: "(SUBQUERY(%K, $x, $x.%K >= %lu).@count == 0)", #keyPath(City.residents), #keyPath(Person.age), 36)
```
## 匹配对象和对象ID
```
let predicate =  NSPredicate(format: "self == %@", object)
let predicate =  NSPredicate(format: "self == %@", objectID)
let predicate = NSPredicate(format: "self IN %@", somePeople)
let predicate = NSPredicate(format: "%K CONTAINS %@", #keyPath(City.visitors), person)
```

## 匹配字符串
```
//[n]表示ASCII编码
//?表示匹配单个任意字符串
//*表示匹配0或多个任意字符
let predicate = NSPredicate(format: "%K ==[n] %@", #keyPath(Country.alpha3Code), "ZAF")
let predicate = NSPredicate(format: "%K BEGINSWITH[n] %@", #keyPath(Country.alpha3Code), "CA")
let predicate = NSPredicate(format: "%K ENDSWITH[n] %@", #keyPath(Country.alpha3Code), "K")
let predicate = NSPredicate(format: "%K CONTAINS[n] %@", #keyPath(Country.alpha3Code), "IN")
let predicate = NSPredicate(format: "%K LIKE[n] %@", #keyPath(Country.alpha3Code), "?A?")
let predicate = NSPredicate(format: "%K MATCHES[n] %@", #keyPath(Country.alpha3Code), "[AB][FLH](.)")
```
## 字符串和索引
==[n], BEGINSWITH[n], 和 IN[n] 这三个谓词操作符都能够使用属性上的索引，ENDSWITH[n], CONTAINS[n], LIKE[n] 和 MATCHES[n] 这几个操作符就无法从索引 上获得收益
## 可转换类型
```
let identifier: NSUUID = someRemoteIdentifier
let predicate = NSPredicate(format: "%K == %@",#keyPath(City.remoteIdentifier), identierfi)
```
City.remoteIdentifier是NSUUID to data，谓词匹配时可以直接传入NSUUID，CoreData会将这个对象转换为二进制数据并且创建相应的 SQLite 查询语句
## 性能和排序表达式
1. 在构建复杂谓词的时候，可以先执行那些简单且 (或) 高性能的部分然后再添加复杂的部分。

例 如有这么一个谓词，它需要同时检查 age 和 carsOwnedCount 这两个属性，但是我们知道只有 age 属性上有一个索引，那么我们可以将检查 age 的这一部分优先放入谓词中:
```
let predicate = NSPredicate(format: "%K > %ld && %K == %ld", #keyPath(Person.age), 32, #keyPath(Person.carsOwnedCount), 2)
```

2. 将最能限制数据集大小的那一部分优先放入谓词

