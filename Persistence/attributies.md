#  Coredata_属性

![attributes](https://github.com/coolboy-ccp/Optimization/blob/master/Persistence/attributes.png)

* Transient, 是否存入持久化存储区
* Indexed, 设置该属性为索引, 提升搜索效率，代价是额外的存储空间
* Validation, 防止非法数据存储到持久化存储区
* Index in Spotlight：这个选项不会影响iOS应用程序，它的用途是把基于CoreData的应用程序通Spotlight集成起来。
* Store in External Record File：启用了该选项之后，系统会把持久化存储区里的数据复制成XML格式，并保存在存储区之外。一般存储较大的二进制文件时开启。SQLite可以直接在数据库里高效的存储不超过100kb的二进制数据
