---
icon: launch
title: 升级到6.8.x
category: 升级到6.8.x
---

## 说明
6.8.x针对6.7.x版本而言有如下些许改变

## 修复
- GetProperty属性获取存在重写属性或者影子属性如new导致获取属性报错修复

## 优化
-  移除UseAutoShardingCreate,框架内部会自动启动线程任务去处理,系统默认的Route里面的`AutoCreateByTime`寓意任然是是否创建表
- `IShardingDbContext`接口移除 `IShardingTransaction`,`ISupportShardingReadWrite`,`ICurrentDbContextDiscover`等接口,添加新方法`IShardingDbContextExecutor GetShardingExecutor()`该方法直接返回`ShardingDbContextExecutor`即可,且仅`ShellDbContext`会被调用
- `IShardingDbContext`移除方法`IVirtualDataSource GetVirtualDataSource();`,`NotifyShardingTransaction`,`Rollback`,`Commit`,`RollbackAsync`,`CommitAsync`,` IDictionary<string, IDataSourceDbContext> GetCurrentDbContexts()`,外加两个滴血分离属性`ReadWriteSeparationPriority`,`ReadWriteSeparation`当然这两个属性用户可以自行选择是否添加,如添加那么还是调用`ShardingDbContextExecutor`内部的相同属性
- `支持CreatePoint`,`RollbackPoint`,`ReleasePoint`



## startup
x.6.x.x+
```csharp

            services.AddShardingDbContext<ShardingDefaultDbContext>()
                .UseRouteConfig(op =>
                {
                    op.AddShardingDataSourceRoute<OrderAreaShardingVirtualDataSourceRoute>();
                    op.AddShardingTableRoute<SysUserModVirtualTableRoute>();
                    op.AddShardingTableRoute<SysUserSalaryVirtualTableRoute>();
                    op.AddShardingTableRoute<OrderCreateTimeVirtualTableRoute>();
                    op.AddShardingTableRoute<LogDayVirtualTableRoute>();
                    op.AddShardingTableRoute<LogWeekDateTimeVirtualTableRoute>();
                    op.AddShardingTableRoute<LogWeekTimeLongVirtualTableRoute>();
                    op.AddShardingTableRoute<LogYearDateTimeVirtualRoute>();
                    op.AddShardingTableRoute<LogMonthLongvirtualRoute>();
                    op.AddShardingTableRoute<LogYearLongVirtualRoute>();
                    op.AddShardingTableRoute<SysUserModIntVirtualRoute>();
                    op.AddShardingTableRoute<LogDayLongVirtualRoute>();
                    op.AddShardingTableRoute<MultiShardingOrderVirtualTableRoute>();

                })
                .UseConfig(op =>
                {
                    //当无法获取路由时会返回默认值而不是报错
                    op.ThrowIfQueryRouteNotMatch = false;
                    //忽略建表错误compensate table和table creator
                    op.IgnoreCreateTableError = true;
                    //迁移时使用的并行线程数(分库有效)defaultShardingDbContext.Database.Migrate()
                    op.MigrationParallelCount = Environment.ProcessorCount;
                    //补偿表创建并行线程数 调用UseAutoTryCompensateTable有效
                    op.CompensateTableParallelCount = Environment.ProcessorCount;
                    //最大连接数限制
                    op.MaxQueryConnectionsLimit = Environment.ProcessorCount;
                    //链接模式系统默认
                    op.ConnectionMode = ConnectionModeEnum.SYSTEM_AUTO;
                    //如何通过字符串查询创建DbContext
                    op.UseShardingQuery((conStr, builder) =>
                    {
                        builder.UseSqlServer(conStr).UseLoggerFactory(efLogger);
                    });
                    //如何通过事务创建DbContext
                    op.UseShardingTransaction((connection, builder) =>
                    {
                        builder.UseSqlServer(connection).UseLoggerFactory(efLogger);
                    });
                    //添加默认数据源
                    op.AddDefaultDataSource("A",
                        "Data Source=localhost;Initial Catalog=ShardingCoreDBA;Integrated Security=True;");
                    //添加额外数据源
                    op.AddExtraDataSource(sp =>
                    {
                        return new Dictionary<string, string>()
                    {
                        { "B", "Data Source=localhost;Initial Catalog=ShardingCoreDBB;Integrated Security=True;" },
                        { "C", "Data Source=localhost;Initial Catalog=ShardingCoreDBC;Integrated Security=True;" },
                    };
                    });

                    //如果需要迁移code-first必须要自行处理
                    o.UseShardingMigrationConfigure(b =>
                    {
                        b.ReplaceService<IMigrationsSqlGenerator, ShardingSqlServerMigrationsSqlGenerator>();
                    });
                    //添加读写分离
                    op.AddReadWriteSeparation(sp =>
                    {
                        return new Dictionary<string, IEnumerable<string>>()
                    {
                        {
                            "A", new HashSet<string>()
                            {
                                "Data Source=localhost;Initial Catalog=ShardingCoreDBB;Integrated Security=True;"
                            }
                        }
                    };
                    }, ReadStrategyEnum.Loop, defaultEnable: false, readConnStringGetStrategy: ReadConnStringGetStrategyEnum.LatestEveryTime);
                })
                .ReplaceService<ITableEnsureManager,SqlServerTableEnsureManager>()
                .AddShardingCore();



            //启动进行表补偿(不调用也可以使用ShardingCore)一般不需要调用比如debug的时候缺表了可以选择调用
            app.ApplictaionServices.UseAutoTryCompensateTable();

```