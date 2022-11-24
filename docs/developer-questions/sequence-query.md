---
icon: mysql
title: 顺序查询
category: 使用疑虑
---

## 全表扫描了

如果你的分片是有序的并且可以通过某些字段排序来进行过滤，sharding-core提供了顺序查询功能,[解决方案](https://www.cnblogs.com/xuejiaming/p/15848454.html)

## 全局提示或阶段防止全表扫描
通过路由后置过滤器
```chsarp
public class SysUserLogByMonthRoute:AbstractSimpleShardingMonthKeyDateTimeVirtualTableRoute<SysUserLogByMonth>
    {
        private readonly ILogger<SysUserLogByMonthRoute> _logger;

        public SysUserLogByMonthRoute(ILogger<SysUserLogByMonthRoute> logger)
        {
            _logger = logger;
        }
        public override DateTime GetBeginTime()
        {
            return new DateTime(2021, 1, 01);
        }

        public override bool AutoCreateTableByTime()
        {
            return true;
        }

        public override void Configure(EntityMetadataTableBuilder<SysUserLogByMonth> builder)
        {
            builder.ShardingProperty(o => o.Time);
        }

        protected override List<TableRouteUnit> AfterShardingRouteUnitFilter(DataSourceRouteResult dataSourceRouteResult, List<TableRouteUnit> shardingRouteUnits)
        {
            if (shardingRouteUnits.Count > 10)
            {
                _logger.LogInformation("截断前:"+string.Join(",",shardingRouteUnits.Select(o=>o.Tail)));
                //这边你要自己做顺序处理阶段
                var result= shardingRouteUnits.OrderByDescending(o=>o.Tail).Take(10).ToList();
                _logger.LogInformation("截断后:"+string.Join(",",result.Select(o=>o.Tail)));
                return result;
            }
            return base.AfterShardingRouteUnitFilter(dataSourceRouteResult, shardingRouteUnits);
        }
    }
```

