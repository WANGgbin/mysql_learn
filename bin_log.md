描述 mysql 中的 bin_log 相关内容。

- bin_log 是干嘛的？

    bin_log 本质上记录了我们对数据库的操作。常见的用途有：
    - 数据恢复
    - 主备同步


- bin_log 存在什么地方

    几个重要的系统变量：

    - `log_bin`

        是否打开 bin log.

    - `datadir`

        bin_log 存放目录

    - `log_bin_basename`

        存放 bin_log 的文件名的 base name

    - `log_bin_index`

        bin log 索引文件名。
    

    几个重要的命令：
    - SHOW BINARY LOGS; 展示所有的 bin log 文件
    - SHOW BINLOG EVENTS IN 'file';展示某一个 bin log 里所有 events

        +----------------------------+-----+----------------+-----------+-------------+-----------------------------------+
        | Log_name                   | Pos | Event_type     | Server_id | End_log_pos | Info                              |
        +----------------------------+-----+----------------+-----------+-------------+-----------------------------------+
        | LAPTOP-K6E7TUKR-bin.000033 |   4 | Format_desc    |         1 |         125 | Server ver: 8.0.26, Binlog ver: 4 |
        | LAPTOP-K6E7TUKR-bin.000033 | 125 | Previous_gtids |         1 |         156 |                                   |
        | LAPTOP-K6E7TUKR-bin.000033 | 156 | Stop           |         1 |         179 |                                   |
        +----------------------------+-----+----------------+-----------+-------------+-----------------------------------+

        bin log 中的每一条记录实际上就是一个 event, pos 表示记录开始的偏移量， end_log_pos 表示记录结束的偏移量。 event_type 表示 event 的类别。 server_id 表示服务器的 id， 在集群模式中，每一个实例要有一个唯一的 id， 即 server_id.

        注意：**该命令无法看到 event 的详细内容，我们可以通过 mysql 自带的 bin log 解析工具来查看 bin log 的详细内容。**
    
    - SHOW MASTER STATUS; 展示当前写入的 bin log 文件。

- bin_log 的格式是怎么样的？有几种格式？不同格式区别是什么？ 应该怎么查看?

    - 三种格式。
      - statement

          即使用 mysql 语句的方式，记录对数据库的变更。
          - 优点：
              
              占用空间小。
          
          - 缺点：
              
              可能会造成主备的数据不一致。比如 执行了一条带 limit 的 del 语句。

              在不同的库中执行时，可能会走不同的索引，而 limit 语句对数量进行了限制，从而导致在不同库上删除的记录不同，从而导致主备不一致。

      - row(默认值)

          将变更对应的所有行信息，记录下来。增删改对应的信息是不一样的，比如改会记录修改前后的信息，删除记录删除前的信息等。

          - 优点：

              不会存在主备不一致的情况，我们也可以通过这种方式来恢复数据。

          - 缺点：

              需要记录所有涉及到的行的内容，会导致 bin log 很大。

      - mixed

          将上述两种方式结合在一起，mysql 会判断使用 statement 是否会导致主备不一致，如果不会则使用 statement， 负责使用 row.

          - 优点：

              兼顾了 statement 和 row 的优点

          - 缺点：

            使用了 statement 格式的 bin_log 无法据此恢复数据。因此：**如果恢复数据很重要的话，总是使用 row 格式**

    - 那么如何修改 bin_log 的格式呢？

        通过 `binlog_format` 来修改 bin log 的格式。

    - 如何通过 `mysqlbinlog` 查看 bin_log 的详细内容呢？

        - 通过指定事务的首个 event 所在的 position 查看 事务的详细信息
            - 首先通过 `SHOW BINLOG EVENTS [IN bin_log_file] [FROM pos] [LIMIT row_count];` 查看对应的 events.
            - 找到 事务所在的首个 event 的 position
            - 通过 `mysqlbinlog -vv bin_log_file --start-position=position` 查看详细内容。

- 如果通过 bin_log 来恢复数据？

    在**ROW**格式下，我们可以用来从指定的位置开始恢复数据，怎么操作呢？
    `mysqlbinlog bin_log_file --start-position=1234 --stop-position=5678 | mysql -h127.0.0.1 -P1234 -u$user -p$pwd`

- 主备同步的流程是什么样的？


- 双 M 架构相比于单 M 架构有什么优势？双 M 架构如何避免循环复制问题？

    双 M 架构指的是，两个主库互为备库，但是任何时候只会写一个主库。
    要实现双 M 架构，要设置 `log_slave_updates = ON` 表示消费完 relay log 后也生成 bin log.

    但是这里有个问题，假设 A -> B, 然后 B -> A, 这不是会导致递归吗？怎么解决呢？还记得每一个 event 都会有一个 server_id 属性，表示真正产生此 event 的节点是哪一个。每一个节点在收到 event 的时候会判断 event 的 server_id 是否跟自己的一样，一样的话就不进行处理。同时 B 转发 A 产生的 bin_log，server_id 保持不变。

- bin_log 大小有限制吗

bin_log 不可能一直追加，如何控制 bin_log 大小，bin_log 的清理策略是啥？

 - 设置单个 bin_log 文件的大小

    通过 `max_binlog_size` 可以设置单个 bin_log 文件的最大容量，默认为 1GB.

 - 设置 bin_log 文件的过期时间

    通过 `binlog_expire_logs_seconds` 可以设置 bin_log 的过期时间，默认为 7Days. 超过此时间的 bin_log 会被清理。


- 主从同步

我们知道可以通过 bin_log 进行主库与从库的同步。如果从库落后主库很多，对应的 bin_log 也被删除了。那么如何同步从库呢？<br>

这个时候