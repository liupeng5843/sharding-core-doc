---
icon: launch
title: 如果不存在就创建表的路由
category: 路由
---


## 自动判断路由

自动判断路由是一种当你需要对某个字段进行分片的时候，比如设备所属区域作为分表的后缀，如果在添加这个设备的时候没有这个表那么就创建这个表，有时候我们可能需要这种路由,下面我们将以mysql作为一个例子进行编写


```csharp

    public class AreaDevice
    {
        public string Id { get; set; }
        public DateTime CreateTime { get; set; }
        public string Area { get; set; }
    }
    
    public class AreaDeviceRoute : AbstractShardingOperatorVirtualTableRoute<AreaDevice, string>
    {
        private readonly IVirtualDataSource _virtualDataSource;
        private readonly IShardingTableCreator _tableCreator;
        private const string Tables = "Tables";
        private const string TABLE_SCHEMA = "TABLE_SCHEMA";
        private const string TABLE_NAME = "TABLE_NAME";

        private const string CurrentTableName = nameof(AreaDevice);
       
        private readonly ConcurrentDictionary<string, object?> _tails = new ConcurrentDictionary<string, object?>(StringComparer.OrdinalIgnoreCase);
        private readonly object _lock = new object();

        public AreaDeviceRoute(IVirtualDataSource virtualDataSource,  IShardingTableCreator tableCreator)
        {
            _virtualDataSource = virtualDataSource;
            _tableCreator = tableCreator;
            InitTails();
        }

        private void InitTails()
        {
            
            using (var connection = new MySqlConnection(_virtualDataSource.DefaultConnectionString))
            {
                connection.Open();
                var database = connection.Database;

                using (var dataTable = connection.GetSchema(Tables))
                {
                    for (int i = 0; i < dataTable.Rows.Count; i++)
                    {
                        var schema = dataTable.Rows[i][TABLE_SCHEMA];
                        if (database.Equals($"{schema}", StringComparison.OrdinalIgnoreCase))
                        {
                            var tableName = dataTable.Rows[i][TABLE_NAME]?.ToString() ?? string.Empty;
                            if (tableName.StartsWith(CurrentTableName,StringComparison.OrdinalIgnoreCase))
                            {
                                //如果没有下划线那么需要CurrentTableName.Length有下划线就要CurrentTableName.Length+1
                                _tails.TryAdd(tableName.Substring(CurrentTableName.Length+1), null);
                            }
                        }
                    }
                }
            }
        }
        

        public override string ShardingKeyToTail(object shardingKey)
        {
            return $"{shardingKey}";
        }
        /// <summary>
        /// 如果你是非mysql数据库请自行实现这个方法返回当前类在数据库已经存在的后缀
        /// 仅启动时调用
        /// </summary>
        /// <returns></returns>
        public override List<string> GetTails()
        {
            return _tails.Keys.ToList();
        }

        public override void Configure(EntityMetadataTableBuilder<AreaDevice> builder)
        {
            builder.ShardingProperty(o => o.Area);
        }

        public override Func<string, bool> GetRouteToFilter(string shardingKey, ShardingOperatorEnum shardingOperator)
        {
            var t = ShardingKeyToTail(shardingKey);
            switch (shardingOperator)
            {
                case ShardingOperatorEnum.Equal: return tail => tail == t;
                default:
                    {
#if DEBUG
                        Console.WriteLine($"shardingOperator is not equal scan all table tail");
#endif
                        return tail => true;
                    }
            }
        }

        public override TableRouteUnit RouteWithValue(DataSourceRouteResult dataSourceRouteResult, object shardingKey)
        { 
            var shardingKeyToTail = ShardingKeyToTail(shardingKey);
            if (!_tails.TryGetValue(shardingKeyToTail, out var _))
            {
                lock (_lock)
                {
                    if (!_tails.TryGetValue(shardingKeyToTail, out var _))
                    {
                       
                        try
                        {
                            _tableCreator.CreateTable<AreaDevice>(_virtualDataSource.DefaultDataSourceName,
                                shardingKeyToTail);
                        }
                        catch (Exception ex)
                        {
                            Console.WriteLine("尝试添加表失败" + ex);
                        }

                        _tails.TryAdd(shardingKeyToTail, null);
                    }
                }
            }

            return base.RouteWithValue(dataSourceRouteResult, shardingKey);
        }
    }
```
整个路由相对较长，这个路由的作用就是启动的时候根据ado.net自动查询表里面有哪些表，然后对其进行分析，获取到这张表目前拥有的分片表，之后对其的值路由进行分析，判断是否没有这个路由如果没有这个表就创建表然后继续执行，这种情况会根据值进行不断地创建表，适合不确定的情况下处理的，但是因为创建表这个动作其实可能会被交由到外部所以需要格外的控制好数据源,这种情况下需要配置一个额外属性必须注意
```csharp
ThrowIfQueryRouteNotMatch = false;
```

