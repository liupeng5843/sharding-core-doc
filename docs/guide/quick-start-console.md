---
icon: launch
title: 快速上手控制台NetCore
category: 使用指南
---


## 快速开始
5步实现按月分表,且支持自动化建表建库
### 第一步安装依赖
`ShardingCore`版本表现形式为a.b.c.d,其中a表示`efcore`的版本号,b表示`ShardingCore`的主版本号,c表示`ShardingCore`次级版本号,d表示`ShardingCore`的修订版本号
```shell
# 请对应安装您需要的版本
PM> Install-Package ShardingCore
# use sqlserver
PM> Install-Package Pomelo.EntityFrameworkCore.MySql
#  use mysql
#PM> Install-Package Pomelo.EntityFrameworkCore.MySql
# use other database driver,if efcore support
```

### 第二步创建查询对象

查询对象
```csharp

    /// <summary>
    /// order table
    /// </summary>
    public class Order
    {
        /// <summary>
        /// order Id
        /// </summary>
        public string Id { get; set; }
        /// <summary>
        /// payer id
        /// </summary>
        public string Payer { get; set; }
        /// <summary>
        /// pay money cent
        /// </summary>
        public long Money { get; set; }
        /// <summary>
        /// area
        /// </summary>
        public string Area { get; set; }
        /// <summary>
        /// order status
        /// </summary>
        public OrderStatusEnum OrderStatus { get; set; }
        /// <summary>
        /// CreationTime
        /// </summary>
        public DateTime CreationTime { get; set; }
    }
    public enum OrderStatusEnum
    {
        NoPay=1,
        Paying=2,
        Payed=3,
        PayFail=4
    }
```
### 第三步创建dbcontext
dbcontext `AbstractShardingDbContext`和`IShardingTableDbContext`如果你是普通的DbContext那么就继承`AbstractShardingDbContext`需要分表就实现`IShardingTableDbContext`,如果只有分库可以不实现`IShardingTableDbContext`接口
```csharp

    public class MyDbContext:AbstractShardingDbContext,IShardingTableDbContext
    {
        public MyDbContext(DbContextOptions<MyDbContext> options) : base(options)
        {
        }

        protected override void OnModelCreating(ModelBuilder modelBuilder)
        {
            base.OnModelCreating(modelBuilder);
            modelBuilder.Entity<Order>(entity =>
            {
                entity.HasKey(o => o.Id);
                entity.Property(o => o.Id).IsRequired().IsUnicode(false).HasMaxLength(50);
                entity.Property(o=>o.Payer).IsRequired().IsUnicode(false).HasMaxLength(50);
                entity.Property(o => o.Area).IsRequired().IsUnicode(false).HasMaxLength(50);
                entity.Property(o => o.OrderStatus).HasConversion<int>();
                entity.ToTable(nameof(Order));
            });
        }
        /// <summary>
        /// empty impl if use sharding table
        /// </summary>
        public IRouteTail RouteTail { get; set; }
    }
```

### 第四步添加分表路由

```csharp
/// <summary>
/// 创建虚拟路由
/// </summary>
public class OrderVirtualTableRoute:AbstractSimpleShardingModKeyStringVirtualTableRoute<Order>
{
    public OrderVirtualTableRoute() : base(2, 3)
    {
    }

    public override void Configure(EntityMetadataTableBuilder<Order> builder)
    {
        builder.ShardingProperty(o => o.Id);
        builder.AutoCreateTable(null);
        builder.TableSeparator("_");
    }
}
```
::: tip 提示
  1. `ShardingProperty`必须指定,表示具体按什么字段进行分表
  2. `AutoCreateTable`可选,表示是否需要在**启动**的时候建表:null表示根据全局配置,true:表示需要,false:表示不需要,默认null
  3. `TableSeparator`可选,表示分表后缀和虚拟表名之间的分隔连接符,默认`_`
  4. `AbstractSimpleShardingModKeyStringVirtualTableRoute<Order>`由sharding-core提供的默认取模分表规则,其中2代表分表后尾巴有两位,3表示按3取模所以后缀为:00,01,02。因为最多2位所以可以最多到99,如果需要了解更多路由[默认路由](/sharding-core-doc/pages/defaultroute)
:::

### 第五步配置今天ShardingCore
创建dbcontext创建者 和静态ShardingRuntimeContext提供者
```csharp
public class ShardingProvider
{
    private static ILoggerFactory efLogger = LoggerFactory.Create(builder =>
    {
        builder.AddFilter((category, level) => category == DbLoggerCategory.Database.Command.Name && level == LogLevel.Information).AddConsole();
    });
    private static readonly IShardingRuntimeContext instance;
    public static IShardingRuntimeContext ShardingRuntimeContext => instance;
    static ShardingProvider()
    {
        instance=new ShardingRuntimeBuilder<MyDbContext>().UseRouteConfig(op =>
            {
                op.AddShardingTableRoute<OrderVirtualTableRoute>();
            })
            .UseConfig((sp,op) =>
            {
                op.UseShardingQuery((con, b) =>
                {
                    b.UseMySql(con, new MySqlServerVersion(new Version()))
                        .UseLoggerFactory(efLogger);
                });
                op.UseShardingTransaction((con, b) =>
                {
                    b.UseMySql(con, new MySqlServerVersion(new Version()))
                        .UseLoggerFactory(efLogger);
                });
                op.AddDefaultDataSource("ds0", "server=127.0.0.1;port=3306;database=console0;userid=root;password=root;");
            }).ReplaceService<IDbContextCreator, MyDbContextCreator>(ServiceLifetime.Singleton).Build();
    }
}


public class MyDbContextCreator:ActivatorDbContextCreator<MyDbContext>
{
    public override DbContext GetShellDbContext(IShardingProvider shardingProvider)
    {
        var dbContextOptionsBuilder = new DbContextOptionsBuilder<MyDbContext>();
        dbContextOptionsBuilder.UseDefaultSharding<MyDbContext>(ShardingProvider.ShardingRuntimeContext);
        return new MyDbContext(dbContextOptionsBuilder.Options);
    }
}
```

### 第六步启动并使用
当然如果你使用new ServiceCollection()来创建的依赖注入那么使用起来和AspNetCore的方式一样
```csharp

using Microsoft.EntityFrameworkCore;
using Sample.ShardingConsole;
using ShardingCore;
using ShardingCore.Extensions;

ShardingProvider.ShardingRuntimeContext.UseAutoShardingCreate();
ShardingProvider.ShardingRuntimeContext.UseAutoTryCompensateTable();

var dbContextOptionsBuilder = new DbContextOptionsBuilder<MyDbContext>();
dbContextOptionsBuilder.UseDefaultSharding<MyDbContext>(ShardingProvider.ShardingRuntimeContext);
using (var dbcontext = new MyDbContext(dbContextOptionsBuilder.Options))
{
    dbcontext.Add(new Order()
    {
        Id = Guid.NewGuid().ToString("n"),
        Payer = "111",
        Area = "123",
        OrderStatus = OrderStatusEnum.Payed,
        Money = 100,
        CreationTime = DateTime.Now
    });
    dbcontext.SaveChanges();
}

Console.WriteLine("Hello, World!");

```
这样所有的配置就完成了你可以愉快地对Order表进行取模分表了

```csharp
[Route("api/[controller]")]
public class ValuesController : Controller
{
        private readonly MyDbContext _myDbContext;

        public ValuesController(MyDbContext myDbContext)
        {
            _myDbContext = myDbContext;
        }

        [HttpGet]
        public async Task<IActionResult> Get()
        {
            var order = await _myDbContext.Set<Order>().FirstOrDefaultAsync(o => o.Id == "2");
            return OK(order)
        }
}
```

```shell
info: Microsoft.EntityFrameworkCore.Database.Command[20101]
      Executed DbCommand (12ms) [Parameters=[@p0='?' (Size = 50), @p1='?' (Size = 50), @p2='?' (DbType = DateTime), @p3='?' (DbType = Int64), @p4='?' (DbType = Int32), @p5='?' (Size = 50)], CommandType='Text', CommandTimeout='30']
      INSERT INTO `Order_02` (`Id`, `Area`, `CreationTime`, `Money`, `OrderStatus`, `Payer`)
      VALUES (@p0, @p1, @p2, @p3, @p4, @p5);
Hello, World!

```
::: tip 提示
  1. 如果程序无法启动请确保一下几点，确认是否已经注入原生的efcore的DbContext,并且在原生的后续对DbContextOptions进行了`UseDefaultSharding<MyDbContext>()`配置
  2. 目前`ShardingCore`提供了三种配置方式
  - 1.默认配置
  ```csharp
  services.AddShardingDbContext<MyDbContext>()
  ```
  这个配置包含了`AddDbContext`+`AddShardingConfigure`
  - 2.额外配置
  ```csharp
    //原来的dbcontext配置
    services.AddDbContext<MyDbContext>(ShardingCoreExtension.UseDefaultSharding<MyDbContext>);
    //额外添加分片配置
    services.AddShardingConfigure<MyDbContext>()
  ```
  - 3.字符串配置适合单配置的情况下
  ```csharp
    //原来的dbcontext配置
    services.AddDbContext<MyDbContext>(options=>options.UseSqlServer("连接字符串必须和AddConfig的DefaultDataSource一样").UseSharding<TodoAppDbContext>());//UseSharding<TodoAppDbContext>()必须要配置
    //额外添加分片配置
    services.AddShardingConfigure<MyDbContext>()
  ```
  3. default data source 的连接字符串是否和默认dbcontext创建的一致
  4. 是否添加了分表路由`op.AddShardingTableRoute<OrderVirtualTableRoute>();`
:::
