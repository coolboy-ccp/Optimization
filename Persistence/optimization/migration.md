# 版本迁移

## 轻量级数据迁移(CoreData自动推断的映射模型)
轻量级数据迁移能够满足以下数据变化:
* add/delete/ rename attributes/entities
* delete/rename relations
* 更改属性的可选状态
* add/delete 属性的索引
* add/delete/change 实体的符合索引
* add/delete/change 实体的唯一性限制

注意点：

1. 将属性从可选改成不可选时，需要指定默认值
2. 更改索引/符合索引时，需要为更改的属性/实体指定Hash Modifier，以确保数据迁移过程中能够做出正确操作。
3. rename实体/关系/属性时，需要设置Renaming ID。注意，每个版本都需要设置

## 复杂数据迁移(自定义映射模型)
复杂数据迁移，如：
* 实体合并
* 实体拆分
* 新增关系

### 自定义映射模型
创建映射模型流程:
1. 添加新版本

![add version](https://github.com/coolboy-ccp/Optimization/blob/master/Persistence/optimization/images/addVersion1.png)
![add version](https://github.com/coolboy-ccp/Optimization/blob/master/Persistence/optimization/images/addVersion2.png)

2. 添加mapping model

![add mapping](https://github.com/coolboy-ccp/Optimization/blob/master/Persistence/optimization/images/addMapping1.png)
![add mapping](https://github.com/coolboy-ccp/Optimization/blob/master/Persistence/optimization/images/addMapping2.png)
![add mapping](https://github.com/coolboy-ccp/Optimization/blob/master/Persistence/optimization/images/addMapping3.png)
![add mapping](https://github.com/coolboy-ccp/Optimization/blob/master/Persistence/optimization/images/addMapping4.png)

### 自定义映射策略
继承NSEntityMigrationPolicy，重写createDestinationInstances


