1. 如何给 DDL 设置过期时间?
可以通过系统变量: `lock_wait_timeout` 设置 mysql 中 DML 语句的过期时间, 默认值是 31536000, 单位 s 即 1年.取值范围: 1 - 31536000
2. 事务中的语句顺序如何安排呢?
3. 为什么尽量要避免大事务?
- 大事务导致对应的 read_view 一直无法释放,从而导致 purge 线程无法回收一些 updaet unod log 以及 真正删除一些记录,从而导致表空间越来越大.
- 主从延迟
- 锁冲突
4. delete 真正的删除数据了吗?
5. 如何查看慢查询日志? https://blog.csdn.net/m0_67394002/article/details/126113577
6. sql 都有哪些数据类型? 以及使用场景是什么?
7. 加锁相关纠正
   - 原文:当隔离级别 >= REPEATABLE READ 时,如果匹配模式不是精确匹配,并且没有找到匹配的记录,则会为该扫描区间后面的一条记录加 next-key 锁.
   - 纠正:实际上,在 mysql8.0.32 中进行测试发现,只加 gap 锁,这可以被认为是一种优化.

8. 如何通过 mysqlbinlog 恢复数据?
mysqlbinlog master.000001  --start-position=2738 --stop-position=2973 | mysql -h127.0.0.1 -P13000 -u$user -p$pwd;
9.  