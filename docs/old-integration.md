---
icon: launch
title: 现有系统接入
category: 现有系统接入
---

## 注意点
现有系统如果你是efcore的需要接入`ShardingCore`需要注意以下几点

### 批处理
默认efcore下的批处理是基于当前DbContext的,如果现有系统接入并且替换了现有DbContext那么需要注意批处理的处理要兼容,ShardingCore默认是ShellDbContext带多个ExecutorDbContext，ShellDbContext会代理原先的查询。所以原先的批处理不能在DbContext上使用,具体参考 [批处理文档](/sharding-core-doc/adv/batch-operate) 不要问为什么,因为批处理不是efcore的接口

### 追踪
默认`ShardingCore`可以支持追踪哪怕是读写分离的情况下,包括select，add，update,remove,但是如果需要获取DbContext.ChangeTracker下的方法那么会无法获取目前有两种办法:
- 1.重写ChangeTracker可以参考源码的`ShardingChangeTracker`,通过替换`IChangeTrackerFactory`
- 2.通过`AbstractShardingDbContext`下的`IShardingDbContextExecutor`获取当前的DbContext然后可以获取到对应的DbContext进行处理

- 3.如果你需要在SaveChange的时候对一些变更信息进行记录或者审计那么可以通过重写带bool参数的SaveChange
方法。
```csharp
        public override int SaveChanges(bool acceptAllChangesOnSuccess)
        {
            if (IsExecutor)
            {
                //审计操作
                //ChangeTracker.Entries().Where(xxxxx)
            }
            
            return base.SaveChanges(acceptAllChangesOnSuccess);
        }
```

