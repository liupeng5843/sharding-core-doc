---
icon: customize
title: 参数配置
category: 使用指南
---

## 配置
```csharp
 
            services.AddShardingDbContext<OtherDbContext>()
                .UseRouteConfig(op => { op.AddShardingTableRoute<MyUserRoute>(); })
                .UseConfig(o =>
                {
                    //当查询无法匹配到对应的路由是否抛出异常 true表示抛出异常 false表示返回默认值
                    o.ThrowIfQueryRouteNotMatch = false;
                    //如果开启读写分离是否在savechange commit rollback后自动将dbcontext切换为写库 true表示是 false表示不是
                    o.AutoUseWriteConnectionStringAfterWriteDb = false;
                    //创建表如果出现错误是否忽略掉错误不进行日志出输出 true表示忽略 false表示不忽略
                    o.IgnoreCreateTableError = false;
                    //默认的系统默认迁移并发数,分库下会进行并发迁移
                    o.MigrationParallelCount = Environment.ProcessorCount;
                    //补偿表并发线程数 补偿表用来进行最后一次补偿缺少的分片表
                    o.CompensateTableParallelCount = Environment.ProcessorCount;
                    //默认最大连接数限制 如果出现跨分片查询需要开启n个链接，设置这个值会将x个链接分成每n个为一组进行同库串行执行
                    o.MaxQueryConnectionsLimit = Environment.ProcessorCount;
                    //链接模式的选择系统自动
                    o.ConnectionMode = ConnectionModeEnum.SYSTEM_AUTO;
                    o.UseShardingQuery((conStr, builder) =>
                    {
                        builder.UseMySql(conStr, new MySqlServerVersion(new Version()))
                            .UseLoggerFactory(efLogger)
                            .EnableSensitiveDataLogging()
                            .UseQueryTrackingBehavior(QueryTrackingBehavior.NoTracking);
                    });
                    o.UseShardingTransaction((connection, builder) =>
                    {
                        builder
                            .UseMySql(connection, new MySqlServerVersion(new Version()))
                            .UseLoggerFactory(efLogger)
                            .EnableSensitiveDataLogging()
                            .UseQueryTrackingBehavior(QueryTrackingBehavior.NoTracking);
                    });
                    o.UseShardingMigrationConfigure(b =>
                    {
                        b.ReplaceService<IMigrationsSqlGenerator, ShardingMySqlMigrationsSqlGenerator>();
                    });
                    o.AddDefaultDataSource("ds0",
                        "server=127.0.0.1;port=3306;database=dbdbdx;userid=root;password=root;");
                    o.AddExtraDataSource(sp => new Dictionary<string, string>()
                    {
                        { "ds1", "server=127.0.0.1;port=3306;database=dbdbd1;userid=root;password=root;" },
                        { "ds2", "server=127.0.0.1;port=3306;database=dbdbd2;userid=root;password=root;" }
                    });
                    o.AddReadWriteSeparation(sp=>new Dictionary<string, IEnumerable<string>>()
                    {
                        {"ds0",new []{"server=127.0.0.1;port=3306;database=dbdbdx;userid=root;password=root;"}}
                    },ReadStrategyEnum.Loop,false,10,ReadConnStringGetStrategyEnum.LatestFirstTime);
                })
                .ReplaceService<ITableEnsureManager, MySqlTableEnsureManager>()
                .AddShardingCore();
```


## ThrowIfQueryRouteNotMatch

`sharding-core`默认会在查询无法匹配到对应的路由是否抛出异常 true表示抛出异常 false表示返回默认值。

## IgnoreCreateTableError

`sharding-core`默认会在创建表失败后输出错误信息,但是输出的信息会被log记录所以为了log不记录这些信息，可以将这个值设置为true那么如果创建失败(已经存在表)框架将不会抛出对应的错误消息，如果您是使用[code-first](/sharding-core-doc/adv/code-first/)的那么这个值可以无视或者设置为false。

## MaxQueryConnectionsLimit

最大并发链接数，就是表示单次查询`sharding-core`允许使用的dbconnection，默认会加上1就是说如果你配置了`MaxQueryConnectionsLimit=10`那么实际`sharding-core`会在同一次查询中开启11条链接最多,为什么是11不是10因为`sharding-core`会默认开启一个链接用来进行空dbconnection的使用。如果不设置本参数那么默认是cpu线程数`Environment.ProcessorCount`

## ConnectionMode
链接模式,可以由用户自行指定，使用内存限制,和连接数限制或者系统自行选择最优

链接模式，有三个可选项，分别是：
### MEMORY_STRICTLY
内存限制模式最小化内存聚合 流式聚合 同时会有多个链接

MEMORY_STRICTLY的意思是最小化内存使用率，就是非一次性获取所有数据然后采用流式聚合

### CONNECTION_STRICTLY
连接数限制模式最小化并发连接数 内存聚合 连接数会有限制

CONNECTION_STRICTLY的意思是最小化连接并发数，就是单次查询并发连接数为设置的连接数`MaxQueryConnectionsLimit`。因为有限制，所以无法一直挂起多个连接，数据的合并为内存聚合采用最小化内存方式进行优化，而不是无脑使用内存聚合


### SYSTEM_AUTO
系统自动选择内存还是流式聚合

系统自行选择会根据用户的配置采取最小化连接数，但是如果遇到分页则会根据分页策略采取内存限制，因为skip过大会导致内存爆炸


::: warning 注意
!!!如果用户手动设置ConnectionMode则按照用户设置的为准,之后判断本次查询skip是否大于UseMemoryLimitWhileSkip,如果是采用`MEMORY_STRICTLY`,之后才是系统动态设置根据`MaxQueryConnectionsLimit`来分配!!!
!!!如果用户手动设置ConnectionMode则按照用户设置的为准,之后判断本次查询skip是否大于UseMemoryLimitWhileSkip,如果是采用`MEMORY_STRICTLY`,之后才是系统动态设置根据`MaxQueryConnectionsLimit`来分配!!!
!!!如果用户手动设置ConnectionMode则按照用户设置的为准,之后判断本次查询skip是否大于UseMemoryLimitWhileSkip,如果是采用`MEMORY_STRICTLY`,之后才是系统动态设置根据`MaxQueryConnectionsLimit`来分配!!!
:::
