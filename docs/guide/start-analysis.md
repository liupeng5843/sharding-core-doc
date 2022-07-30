---
icon: customize
title: 启动解析
category: 使用指南
---
## 启动解析

### 如何让efcore支持分片
如何让efcore支持分片这边就简单说下

efcore的原生使用需要一个`DbContextOptions`也可以是泛型,那么`DbContextOptions`是怎么来的,是通过`DbContextOptionsBuilder`下的`Options`属性来获取的,为了让efcore的`dbcontext`可以支持分片功能我们需要将自己的一些配置信息注入到efcore内部,那么只要我们注入到`DbContextOptionsBuilder`的`Options`这个里面那么efcore内部就可以获取到对应的配置了最简实现
```csharp
IShardingRuntimeContext shardingRuntimeContext = null;
var dbContextOptionsBuilder = new DbContextOptionsBuilder();
dbContextOptionsBuilder.UseSqlServer("").UseDefaultSharding<DefaultShardingTableDbContext>(shardingRuntimeContext);
var defaultShardingTableDbContext = new DefaultShardingTableDbContext(dbContextOptionsBuilder.Options);           
```
通过上述我们可以看到为了扩展efcore我们只需要做到两步即可
- 第一步创建一个`IShardingRuntimeContext`
- 第二部`UseDefaultSharding<DefaultShardingTableDbContext>(shardingRuntimeContext)`

### 如何创建IShardingRuntimeContext
接下来问题来到了如何创建`IShardingRuntimeContext`
```csharp

IShardingRuntimeContext shardingRuntimeContext = new ShardingRuntimeBuilder<DefaultShardingTableDbContext>()
                .UseRouteConfig()
                .UseConfig()
                .Build();
```
这个`ShardingRuntimeBuilder<>`就是最终用来创建`IShardingRuntimeContext`的，并且`IShardingRuntimeContext`的内部是一个依赖注入的服务所以我一将当前应用程序的声明周期的IServiceProvider传入到内部例如

```csharp
IShardingRuntimeContext shardingRuntimeContext = new ShardingRuntimeBuilder<DefaultShardingTableDbContext>()
                .UseRouteConfig()
                .UseConfig()
                .Build(applicationServiceProvider);
```
这样`ShardingCore`内部的所有路由都可以获取到当前应用程序的依赖注入配置信息

### 额外配置
如果你的启动使用的如下的配置,并没有将当前应用程序的依赖注入放入,并且`DbContext`默认也不是依赖注入获取的那么就需要额外实现一个服务`IDbContextCreator`,否则程序将无法正常的进行允许

```csharp

IShardingRuntimeContext shardingRuntimeContext = new ShardingRuntimeBuilder<DefaultShardingTableDbContext>()
                .UseRouteConfig()
                .UseConfig()
                .Build();
```
如果您是依赖注入获取的`DbContext`那么是无需重写`IDbContextCreator`的

### 重写IDbContextCreator
首先我们要定义一个自己的`DbContextCreator`
#### 如果你的`dbcontext`构造函数有且仅有一个并且是`DbContextOptions`的话那么可以这样
```csharp
public class MyDbContextCreator:ActivatorDbContextCreator<MyDbContext>
{
    public override DbContext GetShellDbContext(IShardingProvider shardingProvider)
    {
        IShardingRuntimeContext shardingRuntimeContext =//通过静态类来实现 ShardingProvider.ShardingRuntimeContext
        var dbContextOptionsBuilder = new DbContextOptionsBuilder<MyDbContext>();
        dbContextOptionsBuilder.UseDefaultSharding<MyDbContext>(shardingRuntimeContext);
        return new MyDbContext(dbContextOptionsBuilder.Options);
    }
}
```

`IShardingRuntimeContext`静态类可以这么来实现

```csharp
public class ShardingProvider
{
    private static readonly IShardingRuntimeContext instance;
    public static IShardingRuntimeContext ShardingRuntimeContext => instance;
    static ShardingProvider()
    {
        instance=new ShardingRuntimeBuilder<MyDbContext>()
        .UseRouteConfig()
            .UseConfig()
            .ReplaceService<IDbContextCreator, MyDbContextCreator>()
            .Build();
    }
}
```

#### 如果你的`dbcontext`构造函数有多个的情况下那么可以这么来实现
```csharp
public class MyDbContextCreator: IDbContextCreator
    {
        /// <summary>
        /// 如何创建dbcontext
        /// </summary>
        /// <param name="shellDbContext"></param>
        /// <param name="shardingDbContextOptions"></param>
        /// <returns></returns>
        public virtual DbContext CreateDbContext(DbContext shellDbContext, ShardingDbContextOptions shardingDbContextOptions)
        {
           var outDbContext = (MyDbContext)shellDbContext;
            var dbContext = new MyDbContext(shardingDbContextOptions.DbContextOptions,outDbContext.OtherParams);
            if (dbContext is IShardingTableDbContext shardingTableDbContext)
            {
                shardingTableDbContext.RouteTail = shardingDbContextOptions.RouteTail;
            }
            _ = dbContext.Model;
            return dbContext;
        }

        public virtual DbContext GetShellDbContext(IShardingProvider shardingProvider)
        {
            IShardingRuntimeContext shardingRuntimeContext =//通过静态类来实现 ShardingProvider.ShardingRuntimeContext
            var dbContextOptionsBuilder = new DbContextOptionsBuilder<MyDbContext>();
            dbContextOptionsBuilder.UseDefaultSharding<MyDbContext>(shardingRuntimeContext);
            return new MyDbContext(dbContextOptionsBuilder.Options);
        }
    }
```
## 总结
如果您是依赖注入使用的`ShardingCore`那么基本上只需要配置即可,如果您不是依赖注入的`ShardingCore`除了需要配置外还需要一点就是要告诉`ShardingCore`如果凭空创建一个`ShellDbContext`