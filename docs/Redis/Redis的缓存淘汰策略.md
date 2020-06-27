### Redis的缓存淘汰策略

<br desc/>

#### 过期策略

Redis将定期扫描内存中过期的key，并执行删除操作。

redis.conf中默认配置如下：

```
hz 10  #表示每秒钟执行10次过期key扫描
```

每次扫描都是随机的，也就是说有可能已经过期的没有被随机到，如果在删除之前访问了过期的key，Redis将自动删除该key并返回nil。

<br desc/>

#### LRU内存淘汰机制

当内存使用已满时将触发内存淘汰机制。

maxmemory-policy:

1. volatile-lru: 针对设置了过期时间的key，使用LRU算法移除key 
2. allkeys-lru: 针对所有key，使用LRU算法移除key
3. volatile-random: 针对设置了过期时间的key，随机移除key
4. allkeys-random: 针对所有key，随机移除key
5. volatile-ttl: 针对设置了过期时间的key， 移除过期时间最小的key
6. noeviction: 永不过期(默认)。如果内存已满，写操作会返回错误信息

> L: limit 		最少
>
> R: recently  最近
>
> U: use 		使用
>
> LRU: 最近最少使用的

