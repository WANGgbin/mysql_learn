- 日志格式

    redo log 就是用来记录对哪些磁盘页进行了什么样的改动。似乎很简单，我们只需要记录对哪些磁盘页的什么位置进行了什么样的改动。
    但实际上，一条语句可能修改很多页，比如插入一条新的记录，需要往所有的 B+ 树中插入一条记录，如果遇到页分裂，那么更改的页更多。对于一个具体的页，也会涉及很多内容：包括页统计信息，槽，上一条记录的 next_record 等等。如果每一处改动都需要一条 redo log 的话，就会导致系统中产生大量的 redo log。 那么应该怎么处理该问题呢？

    mysql 思想是保存**最少必要信息，我们可以通过这些信息计算得到其他信息**。这是一个很重要的思想。具体到插入记录场景，我们只需要保存新记录，在重放的时候，执行一遍插入的流程，就可以覆盖所有需要更改的位置。注意 mysql 中的每一条 redo log 对应一个页的更改。

- redo log 怎么保证高性能的？

    我们知道之所以存在 redo log 的原因是，同步刷新更新的磁盘页到内存成本比较高，而且基本是乱序写入。因此，仅仅更改内存中的页，并不会同步刷新到磁盘。带来的问题是，如果服务器异常，修改会丢失。因此引入了 redo_log, 因为 redo_log 是顺序写入的，因此性能会高很多。

    但是，即便如此，仍然存在磁盘 I/O，还是很影响系统的性能，怎么办呢？

    mysql 引入了 log buf 的概念，log buf 本质就是个内存缓冲区，用来存放 redo log。在必要的时候，才会刷盘。这样会有问题吗？ 比如，redo log 写到了 log buf 但是还没有写入到磁盘，系统宕机了，那么 redo log 岂不是丢失了？

    实际上，这不会有什么问题，事务提交之前，如果做的变更丢失了，那就丢失了，相当于事务什么都没做。但是，**如果事务提交了，就会将对应的 redo log 刷新到磁盘中，这样保证了事务的持久性**。那么除了事务提交，还有哪些场景会刷新 redo log 到磁盘呢？

    - 当 log buf 中的内容大于 log buf 一半的时候
    - 定时刷新
    - 正常关机
 
    我们知道 log buf 的大小是有限的，如果 log buf 写满了，再写就会覆盖 log buf 最前面的内容，那怎么办？

    为此，mysql 引入了两个变量：

    - buf_free

        表示下一条 redo log 应该从哪里写入。

    - buf_next_to_write

        表示下一条要刷新到磁盘中的 redo log 的位置。

    这样，在将 redo log 写入到 log buf 中的时候，只要还没有与 buf_next_to_write 重合，就可以放心写入了。

- redo log 文件

    redo log 文件存放在什么地方，可以有几个文件，每个文件的大小是多少？
    这可以分别通过三个系统变量进行设置：

    - innodb_log_group_home_dir
    - innodb_log_file_size
    - innodb_log_files_in_group

    可以看到 log 文件组的大小是有限的，那么如果 log 文件组写满了怎么办？

    实际上，如果 redo log 对应的脏页刷新到磁盘了，那么 redo log 就没有存在的意义了。
    为此，mysql 引入了 lsn(log sequence number) 的概念，即每一条 redo log 都会对应一个 lsn。
    同时，根据页第一次修改的时间，将 buffer pool 中的脏页串联在一个 flush list 中，脏页的控制块中会保存脏页对应的第一条 redo log 开始位置的 lsn。随着脏页不断的刷盘，我们只要获取到 flush 链表末尾(flush 是头插法，越靠近尾部，页修改时间越早)页对应的 lsn，便可知道该 lsn 之前的所有 redo log 都已经无效了。

    为此，系统引入了 checkpoint lsn 的概念，checkpoint lsn 保存在 log 文件组的第一个文件中，表示第一条有效日志开始位置的 lsn，checkpoint lsn 之前的 redo log 已经无效。那么 checkpoint 是如何更新的呢？

    系统中会有一个线程，定时的扫描 flush 链表，检测最后一个脏页保存的 lsn，然后将该 lsn 以及对应的 offset 刷新到磁盘中，这样在写入磁盘的时候，只要未达到该 offset 就可以放心写入了。我们将这个动作称为 checkpoint. 不过，**如果用户写入很频繁，且系统刷新脏页比较慢，就可能出现 redo log 日志文件组满的情况，此时就需要线程同步刷新 flush 链表中的脏页到磁盘，从而可以执行 checkpoint.**

- 服务器宕机重启后，重放 redo log 的详细流程是什么？

    系统重启后，从日志文件组中获取 checkpoint offset, 然后执行每一条 redo log。这里其实有些需要注意的细节：

    - 在执行 checkpoint 之后，可能又有脏页刷新到磁盘了，因此并不是 checkpoint offset 之后的每一条 redo log 都需要重放。
    - 我们知道 redo 描述了页的更改，再重放 redo log 之前，会将所有 redo log 以 space_id + page_no 组成一个哈希表，从而在恢复某个页的时候，可以将该页关联的所有 redo log 一次性执行了。


- 一个重要的系统变量 `innodb_flush_log_at_trx_commit`

    表示事务提交时，应该如何刷新对应的 redo log, 可选值有：
    
    - 0

        不立即刷新 redo log

    - 1

        刷新 redo log 到磁盘

    - 2

        将 redo log 写到操作系统的缓冲中。