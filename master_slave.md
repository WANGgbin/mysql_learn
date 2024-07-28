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

### 主备同步流程

  - 主库执行完事务后，将变更内容写入到 bin_log
  - 主库 dump_thread 发送 bin_log 给从库，从库 io_thread 将bin_log 并写入到 relay_log(中继日志) 中
  - 从库 sql_thread 消费 bin_log

为什么从库要有两个线程呢？只需要一个线程是不是就可以了，从主库读取 bin_log 然后 apply。有以下几方面原因：

- 消除 master 与 slave 生产/消费速率差异，尽可能将更多的 bin_log 发送给 slave

    如果是同步的方式，如果 slave 处理速度慢，导致 master 的 bin_log 无法及时发送给 slave，这个时候，如果 master 宕机了，那就会丢失很多的 bin_log 信息。而如果 slave 专门拉起一个线程
    专门读取&保存 bin_log，这样就可以消除 master 与 slave 生产与消费的速度差异，在 master 产生的 bin_log 的时刻，就尽可能的把 bin_log 发送给 slave。然后 slave 的 sql_thread 可以
    慢慢消费 bin_log


### 主备延迟原因    
   
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


# 复制

复制是为了提高 mysql 系统的高可用性，避免单点故障，同时多个从节点可以分摊读流量，提高系统整体的吞吐。

## 复制原理

通过 bin_log 进行同步。主库怎么知道应该从哪个位置给从库发送 bin_log 呢？<br>

复制通常有两种思路，推和拉。推指的是 master 主动同步数据给 slave，这种方式 master 需要维护 slave 的复制进度。拉指的是 slave 主动从 master 拉取数据，这种方式的缺点是
master 与 slave 同步数据不及时，且在 master 异常退出时，slave 丢失数据更多。<br>

mysql 采用的是推的方式，当我们通过 change master + start slave 拉起一个 slave 的时候，slave 会主动告诉 master 从 bin log 的哪个位置开始同步数据。此后，master 中
与该 slave 对应的 dump_thread，就会不断的 push bin_log 给 slave。<br>

这里有个问题，slave 消费 bin log 的速度有可能是慢于 master 生产 bin log 的速度的，slave 如果采用单线程读取+消费 bin log 的方式，就可能导致大量的 bin log 无法及时
从 master 发送到 slave。这样，在 master 异常退出的时候，就会有很明显的问题：大量的 bin log 的丢失。<br>

为此，slave 采用 io_thread + sql_thread 的方式来解决这个问题，io_thread 专门来读取 bin_log，将 bin log 存放到 relay log 文件中，然后 sql_thread 再从 relay log 中
消费 bin log。

## 复制相关文件

- master.info

    保存备库连接主库所需的信息。


## 复制拓扑

### 常见的复制拓扑

常见的复制拓扑有两种：

- 一主多从

    即一个主库多个从库。这种拓扑简单，很常见。

- 互为主备的双主 + 多从

    为什么要设置互为主备的双主呢？好处就是，在切主的时候，不用再通过 change master 的方式指定主库了。

### 主从切换

主从切换通常分为两种：

- 计划中的切换

    比如，要对主库所在的服务器进行升级啥的，就需要选一个新的主库。

- 意外的切换

    比如，主库宕机等。


主从切换最大的难点就是：**如何获取新主库上合适的二进制日志位置，这样其他备库才可以从和老主库相同的逻辑位置开始复制。**

#### 计划中切换

切换流程如下：

- 从库查看跟主库的同步进度

    如果主从延迟很大的话，停止主库的写，从库再追上主库的时间就比较久，这样，整个服务的不可用时间就比较久。因此只有主从延迟的时间小于一个特定阈值的时候，才可以启动切换的流程。

- 主库停止接受写请求

    之所以停止写请求，是为了保证从库可以追上主库的进度，否则从库可能一直落后主库。<br>

    停止主库写的步骤为：

    - 设置 read_only

        此时，服务器不再接受新的写请求。

    - 杀掉还在进行中的事务

        即使设置了 read_only 选项，**当前已经存在的事务可以继续提交**，因此为了真正结束所有的写入，我们 kill 掉所有打开中的事务。

- 选择一个备库，然后确保备库的数据跟主库一致

- 提升从库为主库

    - stop slave
    - change master to master_host = ''; reset slave;

        这两条语句会断开与从库与主库的复制 tcp 连接，并且丢弃 master.info 里的信息。

- 记录新主库 bin log 最新 位置

    - show master status

    之所以要记录该位置，是因为，后续更改让其他从库的主库指向当前库的时候，需要这个位置。

- 等待其他所有从库也完全跟旧主库同步

- 切换用户流量到新主库

- 通过 change master 将所有其他的从库执行新的主库


#### 意外的切换

意外的切换更加复杂，复杂的原因在于：

- 如果有多个从库，应该选择哪一个从库呢？
- 主库的 bin log 可能丢失
- 其他从库指向新的主库的时候，position 如何指定?


我们依次看每一个问题：

- 多个从库中选择新的主库

我们的思路是，选择复制 bin log 最多的从库为新的主库。具体操作为：执行 show slave status 查看 Master_Log_file/ read_Master_Log_Pos，这两个参数
的含义就是，读取 master bin log 的最新位置。


- 主库 bin log 丢失怎么办

即使设置了 sync_binlog = 1，事务提交时，虽然 bin log 落盘了，如果此时主库宕机，bin log 就会丢失。有一种方法来解决 bin log 丢失的问题：

    - 采用可靠的方式存储二进制日志

        相比于将二进制日志直接存储在服务器的硬盘上，将二进制日志存储在某些分布式设备上，即使主库所在的服务器无法启动，二进制日志还是在的。

- 其他从库如何获取新主库的 position

首先让每个从库执行完 relay log，然后通过 mysqlbinlog 工具查看最后一条事件。然后在新主库上通过 mysqlbinlog + grep 查找对应的事件即可。

## 如何配置复制

- 主库配置

    - server id

        不论是主库还是从库，一定要指定一个唯一的 server id。server id 的一个重要应用场景是：如果 bin log 中 event 的 server id 跟当前服务器的 server id 相同，则不应用该 event.

    - 显示指定 log_bin 绝对路径

        mysql 默认的 log_bin 文件前缀是根据服务器主机名命名的，如果服务器主机名发生变化，新生成的 bin log 文件名也会变化。这不利于 bin log 的管理。同时最好也指定一个绝对路径，
        在所有服务器上该绝对路径最好相同，方便 bin log 管理。

    - 打开 sync_binlog

        避免事务提交，但 bin log 未落盘的问题。


- 从库配置

    - server id

        `server_id = unique id`

    - 显示指定 log_bin 绝对路径

        `log_bin = /path/of/log_bin/log_bin_prefix`

    - 显示指定 relay_log 绝对路径

        `relay_log = /path/of/relay_log`

    - 打开 log_slave_updates

        `log_slave_updates = 1`

    - 打开只读选项

        `read_only = 1`

    - 从库崩溃后，禁止重启

        `skip_slave_start = 1`

    - 同步刷新 master.info

        从库崩溃并重启后，会根据 master.info 中的信息从主库同步 bin log，如果不同步刷新 master.info，就会消费重复的 bin log，进而导致错误的发生。因此最好打开此选项。

        `sync_master_info = 1`

- 从库同步 bin log

    - 指定主库

    ```text

        CHANGE MASTER TO MASTER_HOST = 'XXXX',
        MASTER_USER = 'XXX',
        MASTER_PASSWORD = 'xxx',
        MASTER_LOG_FILE = 'mysql-bin.000001',
        MASTER_LOG_POS = 0;

    ```

    - 启动复制

        start slave;

    - 查看同步状态

        start slave status;

## 复制管理与维护

### 主备延迟

- 如何测量主备延迟

    我们都知道 seconds_behind_master 表示主从延迟，那么 second_behind_master = 0 是不是就表示主从没有延迟呢？<br>

    要解决这个问题，我们需要了解 second_behind_master 的计算逻辑，只有当 slave 消费 relay log 的时候，才会根据服务器当前的时间戳以及 bin log event 中
    的时间戳计算 second_behind_master，因此，如果因为一些原因，导致从库的复制中断，那么 second_behind_master 可能一直等于 0.所以该参数不是很准确。<br>

    那该如何确保主从数据一致呢？<br>
    
    查看主从 bin log 的最后一个 event，如果一致，则可以认为从库追赶上了主库。

## 半同步特性

    半同步指的是至少有一个从库接收到 bin log 后，主库才会给用户发送成功。其工作路径如下：

    - 主库提交事务，发送 bin log 给从库

    - 从库接收到 bin log 后(注意不是从库应用 bin log)，就发送响应给主库

    - 主库接收到从库的响应，给客户端发送响应。

    特别需要注意：

    如果主库在一定时间内都没有接收到从库的响应，会将半同步复制转化为异步复制，继续给客户端发送成功响应。
