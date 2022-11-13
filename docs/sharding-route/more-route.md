---
icon: launch
title: 路由命中过多表
category: 路由
---


## 后置过滤器

我们会有这么一个需求比如需要查询某个数据，但是根据where条件可能会过滤出20+的tail结果，导致结果过多，从而影响效率，那么本次我们将通过设置后置过滤来实现这种情况


```csharp

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
这个路由是判断如果本次查询大于10张表那么就只获取最近的10张表


::: warning 注意
建议针对这种情况添加日志，有利于后期排查错误
:::
