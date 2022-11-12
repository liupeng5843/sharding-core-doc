---
icon: launch
title: 简易扩容的哈希路由
category: 路由
---


## 哈希路由

哈希路由的简单实现版本又叫做大数取模分段，在取模的基础上实现了添加单张表后可以将数据迁移做到最小化。
就是在原先的哈希取模的上面进行再次分段来保证不会再增加一个基数的情况下需要大范围的迁移数据，直接上代码


```csharp
            var stringHashCode = ShardingCoreHelper.GetStringHashCode("123");
            var hashCode = stringHashCode % 10000;
            if (hashCode >= 0 && hashCode <= 3000)
            {
                return "A";
            }
            else if (hashCode >= 3001 && hashCode <= 6000)
            {
                return "B";
            }
            else if (hashCode >= 6001 && hashCode < 10000)
            {
                return "C";
            }
            else
                throw new InvalidOperationException($"cant calc hash route hash code:[{stringHashCode}]");
```

这应该是一个最最最简单的是个人都能看得懂的路由了，将hashcode进行取模10000，得到0-9999，将其分成[0-3000],[3001-6000],[6001-9999]三段的概率大概是3、3、4相对很平均，那么还是遇到了上面我们所说的一个问题，如果我们现在需要加一个基数呢，首先修改路由

```csharp
            var stringHashCode = ShardingCoreHelper.GetStringHashCode("123");
            var hashCode = stringHashCode % 10000;
            if (hashCode >= 0 && hashCode <= 3000)
            {
                return "A";
            }
            else if (hashCode >= 3001 && hashCode <= 6000)
            {
                return "B";
            }
            else if (hashCode >= 6001 && hashCode <= 8000)
            {
                return "D";
            }
            else if (hashCode >= 8001 && hashCode < 10000)
            {
                return "C";
            }
            else
                throw new InvalidOperationException($"cant calc hash route hash code:[{stringHashCode}]");
```
我们这边增加了一个基数针对[6001-9999]分段进行了数据切分，并且将[8001-9999]区间内的表后缀没变，实际上我们仅仅只需要修改五分之一的数据那么就可以完美的做到数据迁移，并且均匀分布数据,后续如果需要再次增加一台只需要针对'A'或者'B'进行2分那么就可以逐步增加基数来缓解压力，且数据迁移的数量随着基数的增加响应的需要迁移的数据百分比逐步的减少，最坏的情况是增加一倍的基数需要迁移50%的数据，相比较之前的最好情况迁移50%的数据来说十分划算，而且路由规则简单易写是个人就能写出来。

## 编写简易扩容的哈希路由


那么我们如何在sharding-core里面编写这个路由规则呢
```csharp

    public class OrderHashRangeVirtualTableRoute:AbstractShardingOperatorVirtualTableRoute<Order,string>
    {
        //如何将sharding key的value转换成对应的值
        protected override string ConvertToShardingKey(object shardingKey)
        {
            return shardingKey.ToString();
        }

        //如何将sharding key的value转换成对应的表后缀
        public override string ShardingKeyToTail(object shardingKey)
        {
            var stringHashCode = ShardingCoreHelper.GetStringHashCode(ConvertToShardingKey(shardingKey));
            var hashCode = stringHashCode % 10000;
            if (hashCode >= 0 && hashCode <= 3000)
            {
                return "A";
            }
            else if (hashCode >= 3001 && hashCode <= 6000)
            {
                return "B";
            }
            else if (hashCode >= 6001 && hashCode <= 10000)
            {
                return "C";
            }
            else
                throw new InvalidOperationException($"cant calc hash route hash code:[{stringHashCode}]");
        }

        //返回目前已经有的所有Order表后缀
        public override List<string> GetAllTails()
        {
            return new List<string>()
            {
                "A", "B", "C"
            };
        }

        //如何过滤后缀(已经实现了condition on right)用户无需关心条件位置和如何解析条件逻辑判断，也不需要用户考虑and 还是or
        protected override Func<string, bool> GetRouteToFilter(string shardingKey, ShardingOperatorEnum shardingOperator)
        {
            //因为hash路由仅支持等于所以仅仅只需要写等于的情况
            var t = ShardingKeyToTail(shardingKey);
            switch (shardingOperator)
            {
                case ShardingOperatorEnum.Equal: return tail => tail == t;
                default:
                {
                    return tail => true;
                }
            }
        }
    }
```

**具体的原理如果不清楚可以参考博客[如何自定义路由](https://www.cnblogs.com/xuejiaming/p/15383899.html)**
