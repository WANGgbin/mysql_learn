描述 mysql 主备同步、读写分离相关内容。

# 主备同步

主备同步，我们需要重点考虑的就两个问题：
- 主备不一致
- 主备延迟

## 主备不一致

引起主备不一致的原因都有哪些呢？

- bin_log 格式
- 其他(TODO)

## 主备延迟

要搞明白主备延迟的原因是什么，我们需要先弄清楚主备同步的流程是什么样的。

- 主备同步流程

  - 主库执行完事务后，将变更内容写入到 bin_log
  - 从库通过网络读取 bin_log 并写入到 relay_log(中继日志) 中
  - 从库内部真正消费 bin_log


- 主备延迟原因    
   
    通过分析主备同步流程，我们知道延迟的原因有以下可能：

    - 网络异常
    - 从库的配置比主库的差
    - 从库压力太大

        主库直接影响业务，所以访问比较克制。而从库只提供读能力，有时候访问就比较随意。当执行一些大批量的读操作的时候，就会占用大量的 cpu 资源，从而影响主从的同步导致主备延时。

        可以通过一主多从的方式来分担读压力。

    - 大事务
      - 大表 DDL

    - 从库消费速度小于主库的生产速度

- 查看当前备库同步延迟

    通过 `SHOW SLAVE STATUS;` 其中 `seconds behind master` 表示同步延迟时间，单位为 秒。

- 主备切换策略

    - 可靠性优先

        可靠性优先指的是读到的数据一定是正确的，当主备还未完全同步的时候，备库是不会提供写能力的。我们来看下可靠性优先的主备切换流程：
        - 检测主备延迟时间是否小于某个特定值
        - 如果是，则设置主库为 readonly. 这一步开始，用户不能写数据库，因此数据库变的不可用。
        - 等待，一直到主备延迟为 0 s
        - 此时，设置从库为可写，并将用户请求流量切换到从库。这一步开始，数据库又变的可用。

        从上述流程可以看出，可靠性优先前提下， 系统的不可用时间依赖于主备延迟。因此只有当主备延迟小于某个特定值的时候，才会尝试切换。否则，不可用时间会很长，这是业务不能忍受的。

    - 可用性优先

        可用性优先指的是优先保证数据库是可以正常读写的，至于结果是否正确，并不关心。这种策略下，首先把从库设置为可写，接着直接把用户的流量切换到从库。

        此时，**存库从在两种写操作**
        - 用户写操作
        - relay_log 写操作

        一般来说，只有当 relay_log 写完之后，再执行用户写操作，这样数据是正确的。但现在这两种操作顺序并不能保证，因此可能会产生数据不一致的情况发生。

    一般而言，我们都优先保证 **数据可靠性**。

- GTID

    - 什么是 GTID ？

        GTID 全称(Global Transaction ID) 即全局事务 id，由两部分组成：server_uuid:gno. server_uuid 表示实例的标识，每一个实例启动的时候，都会初始化一个标识这就是 server_uuid. 每一个事务在提交的时候都会被分配一个序号，这就是 gno。

    - 为什么要有 GTID ？

        GTID 主要用于主备同步，当备库上切换主库的时候，我们怎么知道应该从主库 bin_log 的什么地方开始同步呢？
        每一个实例都会维护一个 GTID 集合，标识当前实例执行的所有事务的集合。在 GTID 模式下，在备库上指定新的主库的时候，就会把自己的 GTID 集合发送给目标主库，目标主库会计算自己执行但是备库还未执行的 GTID 差集，然后从该差集中的第一个 GTID 开始，发送 bin_log 给备库。

    - 怎么使用 GTID ？

        只需要设置 `set gtid_mode = on` 和 `set enforce_gtid_consistency = on` 即可打开 GTID 模式。

    - 常见技巧

        可以通过两种方式来指定 GTID 的生成。
        - gtid_next = automatic

            默认值，gtid_next 由系统分配。

        - gtid_next = 'sepcific_gtid'

            通过设置 gtid_next 为某个特定值，将此 gtid_next 分配给下个提交事务。特别注意：此种方式只能用于一个事务，对于下一个事务，**我们需要把 gtid_next 设置为 automatic，或者设置一个新的值**。

        那么什么场景下，我们需要手动设置 gtid_next  为某个特定值呢？

        有时候，从库从主库同步 bin_log，但是在某个事务上发生了错误，导致从库停止同步。如果我们想忽略这个事务，继续同步后面的事务。假设这个事务的 GTID 为 'aaa-bbb-ccc-ddd:1'，我们可以在从库上执行以下语句：
        ```sql

        set gtid_next = 'aaa-bbb-ccc-ddd:1';
        begin;
        commit; -- 提交一个空的事务，目的仅仅是将 'aaa-bbb-ccc-ddd:1' 加入到自己的 GTID 集合中
        set gtid_next = 'automatic'; -- 需要重新设置为 automatic
        start slave; -- 继续开启同步
        ```

        通过上述语句，从库将这个特定的 gtid 加入到自己的集合中，当继续从主库同步信息的时候，就会忽略上述会导致错误的事务，从而能够继续同步。
        所以，**我们可以通过指定 gtid_next 为某个特定值并提交一个空事务的方式来忽略特定的事务**。

- 在线 DDL

   我们知道 DDL 语句会锁表的 MDL，导致所有关于该表的读写查询都阻塞。所以一般不会直接在主库上执行 DDL 语句，避免影响线上业务。那么我们应该怎么做呢？以下针对于双 M 架构。

   - 主库上关闭同步 bin_log: stop slave.
   - 从库上执行 DDL 语句，执行完毕后获取对应的 GTID
   - 主库上提交一个空的事务，事务对应的 GTID 为上述值。
   - 主库开启 bin_log 同步: start slave.
   - 主备切换
   - 从库上关闭 bin_log 同步: stop slave.
   - 主库桑执行 DDL 语句，并获取对应的 GTID。
   - 从库上通过提交一个空事务，添加上述 GTID。
   - 从库开启 bin_log 同步: start slave.
   - 主备切换


# 读写分离

因为主备延迟，因此当写了主库后，立马从从库读，不一定能读到更新的数据。那么怎么保证能够读到最新的数据呢？

  - 读主库

      将读请求划分，对于必须要读取最新数据的请求，发送到主库。
      实际上，这种方法也是使用最多的方法。 但是在某些特殊的业务场景下，可能所有的请求都需要获取最新的数据，如果都发送到主库的话，就会增大主库的压力，也违背了我们通过“读写分离”减少主库压力的初衷。

  - 等 GTID

      整个操作的流程为：
      - 当写操作完成后，可以获取该事务对应的 GTID (set session_track_gtids = OWN_GTID)
      - 选定一个从库
      - 在读从库的时候，检查当前从库的 GTID 集合中是否有该 GTID。 `select wait_for_executed_gtid_set(gtid, 1)`, 第二个参数为要等待的时间，单位 s.
      - 如果有，读取返回结果，否则从主库读。

  - sleep

    还有一种最 low 的方法就是 sleep() 一定的时间再读从库。这种方法不是很优雅，等的时间短了，数据还未同步。等的时间长了，数据可能早已同步。