
在使用 mysql 的过程中经常会遇到一些慢查询的语句。那么怎么查看当前是否存在慢查询的语句？mysql 判断一个语句是不是慢查询的原理又是什么？

- 如何打开慢查询日志功能？

   ```sql
   set slow_query_log = on
   ```

- 慢查询日志存储到什么地方？

    `SHOW VARIABLES LIKE 'slow_query_log_file';`

- 什么样的查询才算慢查询？

    可以通过参数 `long_query_time` 设置慢查询阈值，单位为 s, 默认为 10s. 超过此时间的操作都算是慢查询。

- 如何查看慢查询日志？

    慢查询的日志是人可读的，存储每一条 sql 语句。但在分析一些统计信息，比如：最慢的几条语句的时候，人力统计耗时耗力。我们可以使用 `mysqldumpslow`。

    mysqldumpslow 的详细使用方式可以参考：(mysqldumpslow)[https://blog.csdn.net/weixin_43823808/article/details/124128861]

- 如何是删除慢查询日志
  - rm -rf log_file
  - mysqladmin -hxxx -Pxxx -uxxx flush-logs slow