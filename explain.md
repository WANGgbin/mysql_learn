# explain

我们知道 server 层有个叫优化器的组件，优化器用来对 sql 语句各种可能的执行方式进行成本分析，然后选择最佳的执行方式。这个最佳的执行方式就是所谓的执行计划。当我们想要 debug 某一条语句是怎么执行的时候，就需要查看该语句对应的查询计划。

那么我们有什么方法来查询 sql 语句对应的执行计划吗？有的，通过在 sql 语句前加关键字`explain`来查看对应的执行计划。

接下来，我们详细描述 explain 输出中，每一个列的含义(这里只列举一些需要注意的点，其他的内容可以参考《Mysql 到底是怎么运行的》)

- 每一个表对应一条记录
- type

    type 表示单表的查询类型是什么，比如 all、index、range、const、ref 等。

    - unique_subquery

        当子查询被优化为 EXISTS 语句时，如果子表可以使用主键/唯一二级索引查询，则子表的 type 就是 unique_subquery.

    - index_subquery

        同上，如果子表可以使用普通二级索引查询，则子表的 type 就是 index_subquery.

- ref

    ref 表示的是如果某个表使用二级索引的等值比较的时候，与该二级索引比较的值是什么。可选的值有：const(常数), table.field(某个表的字段), func(函数)

- filtered

    该字段主要用于连接查询的驱动表，表示驱动表的扫描行(rows) 中有多少行是要要进行 join 的。

- key_len

    该字段主要为了让我们在使用联合索引查询的时候，查看优化器使用的前缀是什么。key_len 表示使用索引列长度，单位为字节。比如有两个列，每个列最大长度为 100B， 那么如果 key_len = 100， 我们就知道只使用了联合索引的第一列，如果 key_len = 200, 我们就知道使用了整个联合索引。

    > 这里举的例子中，key_len 长度跟优化器真正计算的长度不完全相同。

- extra

    该字段表示一些额外信息。

    - using index condition

        使用了索引下推

    - using filesort

        使用内存/磁盘进行了排序

    - using temprary

        使用了临时表，比如去重。

    - using join buffer

        连接查询使用了基于块的循环嵌套算法

# optimizer trace

我们通过 explain 只能知道优化器生成的执行计划是什么，但是为什么优化器生成的执行计划是这样的呢？这是一个黑盒，我们是不知道的。

好在 mysql 提供了 optimizer trace 的功能，即可以追踪优化器生成执行计划的详细流程。

- 怎么使用

    mysql 通过系统变量 `optimizer_trace` 控制是否打开 optimizer trace 功能。 默认值是关闭的，因此我们需要先打开此功能。

- trace 信息在哪里

    trace 信息存储在系统库 `information_schema` 的表 `OPTIMIZER_TRACE` 中，query 字段表示 sql 语句，trace 表示 json 格式的执行计划生成内容。

- 怎么查看 trace 信息

    对于单表，描述了可能使用的索引以及每一种方式对应的成本。
    对于 join， 描述了不同的驱动表的成本。

    关于具体如何查看，可以参考 《Mysql 到底是怎么运行的》










