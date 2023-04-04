描述 mysql 中是如何维护表的统计信息的。
两个问题：
- 表的统计信息都有哪些字段，存储在哪里？

    每个表都有两个维度的统计信息，一个是表维度的，一个是索引列维度的。默认存储在 mysql 库的两张表中，分别为 `innodb_table_stats` 和 `innodb_index_stats`。
    
    innodb_table_stats 主要包含的字段有：
    - database_name 数据库名
    - table_name 表名
    - n_rows 表中统计的行数

        通过采样部分页，计算页中记录的平均数量，再结合聚簇索引全部的叶子页数量，来计算表中记录的数量。

    - clustered_index_size 聚簇索引占用的页的数量
    - sum_of_other_index_sizes 其他索引占用的页的数量

        上述两个字段都是通过定位表空间中对应的段(INODE)，再分别统计各个段中使用的页的数量，来计算。

    innodb_index_stats 主要包含的字段有：
    - database_name
    - table_name
    - index_name

        索引名。

    - stat_name
      
      索引对应的统计项名。

      - n_diff_pfx01

        索引第一列的不重复值的个数。对于主键索引和唯一二级索引，只有这一项。

      - n_diff_pfx02
      
        对于非联合索引，表示索引列+主键的不重复值的个数。对于联合索引，表示前两列的不重复值的个数。

      - n_leaf_pages

        索引叶子页的数量。

      - size

        索引总共占用的页面的数量

    - stat_value

        stat_name 对应的值

- 统计信息是怎么统计的？

    表的统计信息是异步进行的。当增、删、改的记录数超过 10% 的时候，就会触发一次异步的表信息统计。

- innodb_stats_method

    当统计索引列的不重复值的时候，如何处理 NULL 值呢？ mysql 提供了一个系统变量 ``, 可选的值有：
    - nulls_equal 所有的 NULL 都是相等的
    - nulls_unequal 每一个 NULL 都是不相等的
    - nulls_ignored 忽略