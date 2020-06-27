### MySQL悲观锁与乐观锁

#### 悲观锁

```mysql
//开启事务
begin;

//查询商品信息
select num from items where id = 1 for update;

//修改商品库存
update items set num = 2 where id = 1;

//提交事务
commit;
```

> 要使用悲观锁需要将MySQL的自动提交属性：
>
> set autocommit=0;

悲观锁利用了MySQL的锁机制，不过行级锁是基于索引的，如果一条SQL没有用到索引，就会使用表级锁将整张表锁住。



#### 乐观锁

**1.利用版本号控制**

```mysql
select version from items where id = 1;

update items set num = 2,version = 2 where id = 1 and version = 1;
```

每次更新就更新递增一次version，不过这种方式会造成并发线程中只有一个会执行成功，其他的都会失败。



**2.利用执行的原子性**

```mysql
update table set num = num -1 where id = 1 and num -1 > 0
```

单条语句执行具有原子性，可以极大的提升吞吐率，提高并发能力