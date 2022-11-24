---
icon: mysql
title: 时间分表问题
category: 使用疑虑
---

## 表并不会自动创建

首先我们要确认你说的`时间分表`是不是我们说的`时间分表`，我们所说的时间分表是设置一个开始时间,然后到现在为止，表会定时进行创建，比如按天会在前一天凌晨创建表。

如果你是想插入某个时间就自动创建那个时间的表你这个叫做`动态分表`，而不是`时间分表`具体的动态分表可以[参考源码demo](https://github.com/dotnetcore/sharding-core/tree/main/samples/Sample.AutoCreateIfPresent) https://github.com/dotnetcore/sharding-core/tree/main/samples/Sample.AutoCreateIfPresent


## 时间分表什么是时候创建表,原理是什么
### 什么时候创建表
默认的时间分表会在对应路由的设置[时间节点进行尝试创建表](/sharding-core-doc/sharding-route/default-route/)

### 原理是什么
默认ShardingCore会在DbContext第一次实例化的时候开启定时任务，定时任务会按照最多每30秒一次扫描内存任务是否满足创建表的需求，如果满足就会调用定时任务通过efcore创建表

### 我按时间分表但是交互是id怎么办
我们是按照时间来进行分表的，那么如果查询条件有时间那么可以精确路由到表，如果没有时间那么将会全表扫描查询，
那么我们系统是id交互的如何才可以保证id交互也可以索引到对应的表
- [解决方案1](https://www.cnblogs.com/xuejiaming/p/15728340.html)  将id设置为雪花id，或者id单纯的添加时间信息，只需要可以通过id解析出表坐落的信息即可，通过sharding-core提供的多字段分片功能
- [解决方案2](https://www.cnblogs.com/xuejiaming/p/15970012.html) 如果你的id是guid无序的那么可以通过文中提供的引入redis来实现辅助功能

### 分页问题
- [解决方案1](https://www.cnblogs.com/xuejiaming/p/15237878.html) 通过配置来解决分页的性能低下
- [解决方案2](https://www.cnblogs.com/xuejiaming/p/15966501.html) 瀑布流分页限制跳页限制排序时间