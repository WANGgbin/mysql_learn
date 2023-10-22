- count 是什么

    count(expr) 是 mysql 中的聚合函数，用来统计符合条件的记录中， expr 不为 NULL 的数量。

- count(*), count(1), count(column) 有区别吗？

    实际上，count(*) 底层就是 count(0). count(\*) 与 count(1) 一样，表示符合条件的记录的数量。
    而 count(column) 只有当 column 不为 NULL 的时候，才会被统计进去。

    另外，如果是统计全表的记录，count(*)/count(1) 都是可以通过 index 的方式，来优化查询的。而 count(column) 如果在 column 上面没有索引，则只能走主键索引的全表扫描，性能会差一些。

- count 底层执行原理

    存储引擎获取一条记录之后交给 Server， Server 会判断这条记录是否满足条件以及对应的 expr 是否不为 NULL， 如果满足条件，则 count++.
    注意：**count 是在 Server 层维护的**

- 如何优化 count(*)

    既然 count(*) 是全表扫描，如果业务中非要统计表的行数应该怎么办呢？两个方法：
    - 缓存系统维护计数

        对数据库有插入/删除操作的时候，同步更新缓存系统中的值。但是有两个问题：
        - 缓存系统可能会导致计数丢失
        - 缓存系统跟数据库的逻辑不一致性，详细可以参考 《mysql 45讲》


    - 数据库中维护计数

    上述问题，可以通过数据库中建立一个单独的计数表来解决问题。