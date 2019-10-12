#  Coredata_关系
![attributes](https://github.com/coolboy-ccp/Optimization/blob/master/Persistence/relationships.png)
* Transient, 是否持久化
* Inverse, 设置逆向关系
* Delete Rule, 删除规则
* Arrangement, 是否排序
* Type, 关系类型
## Type
* to-one
* to-many
* many-to-many
## Delete Rule
![rule](https://github.com/coolboy-ccp/Optimization/blob/master/Persistence/rule.png)
* Deny
   如果关系目标里还有对象，则无法删除当前对象
* Nullify
   将删除对象从它的反向关系里删除。
* cascade
   删除所有与删除对象相关联的对象
* no action
   Coredata不会更新反向关系, 由开发者自定义更新代码
* 自定义更新代码
```
override func prepareForDeletion() {
    ...
}
```
