---
icon: launch
title: Code First
category: 高级
---

## 介绍
用过efcore的小伙伴肯定都知道code first是很久之前就一直在主打的一种编程方式,他可以让我们直接上手编程,而不需要去构建数据库,以一种先写代码后自动创建数据库的模式让开发者从数据库设计中脱离出来，更多的是快速进入到开发的一种状态。

本项目[demo示例](https://github.com/xuejmnet/sharding-core/tree/main/samples/Sample.Migrations)

## 安装
首先无论你是aspnetcore还是普通的控制台程序，我们这边需要做的是新建一个控制台程序，命名为`Project.Migrations`,如果您是分层架构，那安装`Microsoft.EntityFrameworkCore.Tools`请选择对应的`efcore`对应版本

```shell
dotnet add package Microsoft.EntityFrameworkCore.Tools
```

## 开始
首先efcore分为以下几步
首先我们创建几个对象分别是分表和部分表的对象，分表对象又分取模和时间分表
### 创建对象
如果是项目引入efcore层可以忽略
```csharp

    public class NoShardingTable
    {
        public string Id { get; set; }
        public string Name { get; set; }
        public int Age { get; set; }
    }
    
    public class ShardingWithDateTime
    {
        public string Id { get; set; }
        public string Name { get; set; }
        public int Age { get; set; }
        public DateTime CreateTime { get; set; }
    }
    
    public class ShardingWithMod
    {
        public string Id { get; set; }
        public string Name { get; set; }
        public int Age { get; set; }
        public string TextStr { get; set; }
        public string TextStr1 { get; set; }
        public string TextStr2 { get; set; }
    }
```

### 创建数据库关系
如果是项目引入efcore层可以忽略
```csharp

    public class NoShardingTableMap : IEntityTypeConfiguration<NoShardingTable>
    {
        public void Configure(EntityTypeBuilder<NoShardingTable> builder)
        {
            builder.HasKey(o => o.Id);
            builder.Property(o => o.Id).IsRequired().IsUnicode(false).HasMaxLength(128);
            builder.Property(o => o.Name).HasMaxLength(128);
            builder.ToTable(nameof(NoShardingTable));
        }
    }
    public class ShardingWithDateTimeMap : IEntityTypeConfiguration<ShardingWithDateTime>
    {
        public void Configure(EntityTypeBuilder<ShardingWithDateTime> builder)
        {
            builder.HasKey(o => o.Id);
            builder.Property(o => o.Id).IsRequired().IsUnicode(false).HasMaxLength(128);
            builder.Property(o => o.Name).HasMaxLength(128);
            builder.ToTable(nameof(ShardingWithDateTime));
        }
    }
    public class ShardingWithModMap : IEntityTypeConfiguration<ShardingWithMod>
    {
        public void Configure(EntityTypeBuilder<ShardingWithMod> builder)
        {
            builder.HasKey(o => o.Id);
            builder.Property(o => o.Id).IsRequired().IsUnicode(false).HasMaxLength(128);
            builder.Property(o => o.Name).HasMaxLength(128);
            builder.Property(o => o.Name).HasMaxLength(128);
            builder.Property(o => o.TextStr).IsRequired().HasMaxLength(128).HasDefaultValue("");
            builder.Property(o => o.TextStr1).IsRequired().HasMaxLength(128).HasDefaultValue("123");
            builder.Property(o => o.TextStr2).IsRequired().HasMaxLength(128).HasDefaultValue("123");
            builder.ToTable(nameof(ShardingWithMod));
        }
    }
```
### 创建dbcontext
创建dbcontext并且建立关系
```csharp

    public class DefaultShardingTableDbContext:AbstractShardingDbContext, IShardingTableDbContext
    {
        public DefaultShardingTableDbContext(DbContextOptions options) : base(options)
        {
        }

        protected override void OnModelCreating(ModelBuilder modelBuilder)
        {
            base.OnModelCreating(modelBuilder);

            modelBuilder.ApplyConfiguration(new NoShardingTableMap());
            modelBuilder.ApplyConfiguration(new ShardingWithModMap());
            modelBuilder.ApplyConfiguration(new ShardingWithDateTimeMap());
        }

        public IRouteTail RouteTail { get; set; }
    }
```

### 分表路由
如果引用的层里有对应的可以忽略
```csharp

    public class ShardingWithModVirtualTableRoute : AbstractSimpleShardingModKeyStringVirtualTableRoute<ShardingWithMod>
    {
        public ShardingWithModVirtualTableRoute() : base(2, 3)
        {
        }

        public override void Configure(EntityMetadataTableBuilder<ShardingWithMod> builder)
        {
            builder.ShardingProperty(o => o.Id);
        }
    }
    public class ShardingWithDateTimeVirtualTableRoute : AbstractSimpleShardingMonthKeyDateTimeVirtualTableRoute<ShardingWithDateTime>
    {
        public override DateTime GetBeginTime()
        {
            return new DateTime(2021, 9, 1);
        }

        public override void Configure(EntityMetadataTableBuilder<ShardingWithDateTime> builder)
        {
            builder.ShardingProperty(o => o.CreateTime);
        }
    }
```
### SqlGenerator

我们需要对迁移sql生成进行重写,假如我们是`SqlServer`,`MySql`亦是如此

`Oracle`比较特殊可以看源码`Sample.OracleIssue`

```csharp

    public class ShardingSqlServerMigrationsSqlGenerator: SqlServerMigrationsSqlGenerator where TShardingDbContext:DbContext,IShardingDbContext
    {
        private readonly IShardingRuntimeContext _shardingRuntimeContext;

        public ShardingSqlServerMigrationsSqlGenerator(IShardingRuntimeContext shardingRuntimeContext,MigrationsSqlGeneratorDependencies dependencies, ICommandBatchPreparer commandBatchPreparer) : base(dependencies, commandBatchPreparer)
        {
            _shardingRuntimeContext = shardingRuntimeContext;
        }
        protected override void Generate(
            MigrationOperation operation,
            IModel model,
            MigrationCommandListBuilder builder)
        {
            var oldCmds = builder.GetCommandList().ToList();
            base.Generate(operation, model, builder);
            var newCmds = builder.GetCommandList().ToList();
            var addCmds = newCmds.Where(x => !oldCmds.Contains(x)).ToList();

            MigrationHelper.Generate(_shardingRuntimeContext, operation, builder, Dependencies.SqlGenerationHelper, addCmds);
        }
    }
```
```csharp
 services.AddShardingDbContext<DefaultShardingDbContext>()
                .UseRouteConfig(o =>
                {
                    //...
                }).UseConfig(o =>
                {
                    //注意添加这个替换掉默认的
                   
                    o.UseShardingMigrationConfigure(b =>
                    {
                        b.ReplaceService<IMigrationsSqlGenerator, ShardingSqlServerMigrationsSqlGenerator>();
                    });
                })
                .AddShardingCore();



        public void Configure(IApplicationBuilder app, IWebHostEnvironment env)
        {
            if (env.IsDevelopment())
            {
                app.UseDeveloperExceptionPage();
            }

            //建议补偿表在迁移后面
            using (var scope = app.ApplicationServices.CreateScope())
            {
                var myDbContext = scope.ServiceProvider.GetService<MyDbContext>();
                //如果没有迁移那么就直接创建表和库
                myDbContext.Database.EnsureCreated();
                //如果有迁移使用下面的
                // myDbContext.Database.Migrate();
            }
            app.ApplicationServices.UseAutoTryCompensateTable();
            app.UseRouting();

            app.UseAuthorization();

            app.UseEndpoints(endpoints =>
            {
                endpoints.MapControllers();
            });
        }
```


### 迁移初始化命令
```shell
PM> Add-Migration InitSharding
```
如果控制台出现迁移文件就说明迁移成功

### 更新到数据库
```shell
PM> update-Database
```

### 生产环境
```shell
PM> Script-Migration
```
如果是生产环境更多的是通过生成脚本来进行手动处理执行

具体可以参考[Sample.Migrations](https://github.com/xuejmnet/sharding-core/tree/main/samples/Sample.Migrations)