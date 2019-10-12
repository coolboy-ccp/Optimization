#  Coredata
## 架构
![Coredata架构](https://github.com/coolboy-ccp/Optimization/blob/master/Persistence/Coredata架构.png)
* NSManagedObjectModel, 托管对象模型，映射实体类和数据库数据的关系，本质是XML文件
* NSManagerObject, 托管对象，对应数据库数据的实体
* NSManagedObjectContext, 托管对象上下文，管理托管对象
* NSPersistentStoreCoordinator, 持久化存储调度器，用来处理持久化数据和实体类对象的相互转化
* NSPersistentStore, 持久化存储器，负责持久化数据的存取
* NSPersistentContainer, 持久化容器，负责设置model，context，store coordinator
NSPersitsntContainer和其他类的关系参考[官方文档](https://developer.apple.com/documentation/coredata/core_data_stack).

## 创建
1. [创建model](https://developer.apple.com/documentation/coredata/creating_a_core_data_model)
2. [设置container](https://developer.apple.com/documentation/coredata/setting_up_a_core_data_stack)
3. [数据模型](https://developer.apple.com/documentation/coredata/modeling_data)
4. [添加实体](https://developer.apple.com/documentation/coredata/modeling_data/configuring_entities)
5. [添加属性](https://developer.apple.com/documentation/coredata/modeling_data/configuring_attributes)
6. [添加关系](https://developer.apple.com/documentation/coredata/modeling_data/configuring_relationships)
7. [生成类](https://developer.apple.com/documentation/coredata/modeling_data/generating_code)
## 右侧属性栏说明
* 实体
* 属性
* ![attributes](https://github.com/coolboy-ccp/Optimization/blob/master/Persistence/attributes.png)
* 关系
