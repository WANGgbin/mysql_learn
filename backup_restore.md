描述 mysql 中如何备份及恢复数据。

# 备份

## 物理备份

物理备份指的是将 db 文件备份一份。

- 优点

    -  备份快、恢复快。

- 缺点

    - 备份以及恢复容易出错。


### 物理备份方式

- LVM
  
  一种文件系统快照技术，通过 Copy On Write 实现。

## 逻辑备份

逻辑备份指的是将 db 内容导出为 sql 文件。

- 优点

    - 恢复比较简单

        mysqldump 通过管道结合 mysql 即可恢复数据。

    - 可以跨网络恢复

    - 与存储引擎无关

- 缺点

    - 备份，会对 mysql server 造成一定压力。

    - 恢复也可能比较慢。


### 逻辑备份实现

- mysqldump

    mysqldump 将数据库数据导出为 sql 语句。常见的选项包括：

    - 

# 恢复

恢复的整体思路是：**全量备份 + 增量日志**。<br>

一般为了降低服务的不可用时间，**我们不直接在主库上恢复数据，而是会搭建一个临时库**，在临时库上恢复问题语句之前的内容，然后将丢失的数据复制到主库中。而有两种方式搭建临时库：

- 使用 mysqlbinlog 工具重放 binlog
    缺点：
    - 比较慢，mysqlbinlog 只能单线程应用 bin log
    - 通过 --stop-position 方式避免执行有问题语句的位置，不方便。
    - mysqlbinlog 不灵活，最小只能指定数据库粒度，不能指定数据表维度。

- 通过复制的方式重放 binlog
    优点：
    - 快，可以通过并行的方式重放 binlog
    - 通过 replicate-do-table 方式指定表:change replication filter replicate_do_table = (tbl_name)
    - 通过 start slave until 的方式，避免执行有问题的语句。


## 基于复制恢复

不管是 mysqlbinlog 还是 复制，本质都是应用 binlog 的流程。因此，而且复制相对于 mysqlbinlog 恢复过程，性能更好，因此，我们可以通过复制的方式进行恢复。

### 用于快速恢复的延时复制

可以专门搭建一个指定延时的从库，CHANGE MASTER TO MASTER_DELAY = N 命令，可以指定这个备库持续保持跟主库有 N 秒的延迟。<br>

这样在有问题的语句应用到从库之前，先 stop slave。然后定位有问题语句所在的 bin log 位置，然后执行 start slave until 重放问题语句之前的所有语句。然后通过 change master 跳过有问题的语句，再 start slave。然后提升从库为主库。<br>

除了提升主库带来的不可用外，其他流程没有中断服务。

### 使用日志服务器进行恢复

我们可以使用备份搭建一个临时库，然后再用备份的 bin log 创建一个日志服务器(只有 bin log 没有实际数据的 mysql 实例)，然后将临时库设置为日志服务器的从库，并指定 bin log pos。
同时需要定位有问题的语句，然后执行 start slave until 应用问题语句之前的所有 bin log。此时从库就有了丢失的数据。<br>

接着，我们就可以把丢失的内容(表、行)从临时库复制到主库上：

- 复制表

    - 小表

        使用 mysqldump 导出，然后应用到主库。

    - 大表

        使用 “可传输表空间机制(mysql5.6 引入)” 物理拷贝表。注意，直接拷贝表是不会生效的，因为表信息没有注册到系统表中，db 不认识此表。
        关于可传输表空间机制的详情参考：https://time.geekbang.org/column/article/81925

- 复制行

    对于指定行，在临时库执行，select 语句，然后再主库上 insert。