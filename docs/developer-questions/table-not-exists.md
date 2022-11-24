---
icon: mysql
title: 表为什么没有创建
category: 使用疑虑
---

## 我的表为什么没有创建

### 首先确认你使用的路由
如果你是时间分片的路由设置好`GetBeginTime`后将`AutoCreateTableByTime`设置为true那么那么默认路由会在指定的时间去创建对应的表，
但是你可能会发现启动的时候为什么没有表，那么你应该需要考虑自己是否是code-first,并且是否配置了`UseShardingMigrationConfigure`，并且本次迁移migration是否包含了这张表的`create`命令，如果包含了那么`create table`的命令会被分成`N`个，如果没有那么启动时不会给你创建表的，这个时候我们应该怎么办

- 手动创建数据库:最笨的方法就是手动创建数据库.
- 使用补偿表`app.ApplicationServices.UseAutoTryCompensateTable();`这个方法会帮助你扫描缺少的分片表然后给你创建，如果AddShardingDbContext有多个记得挨个获取IShardingRuntimeContext然后按个调用(后续会增加泛型方法)


