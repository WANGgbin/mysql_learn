- limit 用途

    从符合条件的记录中，只获取 n 条数据。一个常见的用途就是 **分页**。 limit baseIndex, count. 表示从第 baseIndex(从 0 开始) 条符合条件的记录中，查找 count 条记录。

- limit 底层原理

    特别需要注意的是，**limit 这个逻辑是在 Server 层进行的**， 当存储引擎给 Server 层发送一条记录后，Server 会判断这条记录是否满足条件，如果满足条件，在发送给客户端之前，还会判断是否满足 limit 语句，满足后，才会发送给客户端。

- limit 如何优化

    一些场景下，limit 语句的性能可能会很差。考虑下面的例子：
    ```sql
    create table test (
        id int not null auto_increment,
        key1 varchar(10) not null,
        primary (id),
        index idx_key1 (key1)
    )engine=innodb;
    ```

    我们进行下面的查询：
    ```sql
    select * from test order by key1 limit 1000, 1;
    ```

    上述语句会怎么执行呢？可能两种方式：
    - 走索引

        走 idx_key1，对于每一个记录回表4，查找到完整的记录后发送给 Server 层。

    - 全表扫描

        因为走索引都需要回表，因此代价可能会比较大，优化器可能会选择全表扫描的方式，扫描所有满足条件的记录。然后在 Server 层进行 filesort 排序，然后再通过 limit 选择出符合条件的记录发送给客户端。

    那么该如何优化上述的语句呢？
    ```sql
    select * from test as t1 inner join (select id from test order by key1 limit 1000, 1) as t2 on t1.id = t2.id;
    ```
    首先，子查询通过索引覆盖可以查询到所有信息，无需回表。然后，两表 join 的时候，将子查询物化表作为驱动表，通过主键访问外层表，大大提高性能。
