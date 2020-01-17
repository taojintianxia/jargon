## MySQL 中有六种日志文件，分别是：
  - 重做日志（redo log）
  - 回滚日志（undo log）
  - 二进制日志（binlog）
  - 错误日志（errorlog）
  - 慢查询日志（slow query log）
  - 一般查询日志（general log）
  - 中继日志（relay log）

其中 `重做日志` 用来保证事务的持久性，`回滚日志` 用来保证事务的原子性以及 InnoDB 的 MVCC。`二进制日志` 也与事务操作有一定的关系。这三种日志，对理解 MySQL 中的事务操作有着重要的意义。

### 重做日志 Redo Log
 
和大多数关系型数据库一样，InnoDB 记录了对数据文件的物理更改，并保证总是日志先行，也就是所谓的 WAL，即在持久化数据文件前，保证之前的 redo log 已经写到磁盘。

LSN(log sequence number) 用于记录日志序号，它是一个不断递增的 unsigned long 类型整数。在 InnoDB 的日志系统中，LSN 无处不在，它既用于表示修改脏页时的日志序号，也用于记录checkpoint，通过 LSN，可以具体的定位到其在 redo log 文件中的位置。

InnoDB的redo log可 以通过参数 `innodb_log_files_in_group` 配置成多个文件，另外一个参数 `innodb_log_file_size` 表示每个文件的大小。因此总的 redo log 大小为：

```
innodb_log_files_in_group * innodb_log_file_size
```
Redo log文件以 `ib_logfile[number]` 命名，日志目录可以通过参数 `innodb_log_group_home_dir` 控制。Redo log 以顺序的方式写入文件文件，写满时则回溯到第一个文件，进行覆盖写。（但在做redo checkpoint 时，也会更新第一个日志文件的头部 checkpoint 标记，所以严格来讲也不算顺序写）。

![](https://raw.githubusercontent.com/taojintianxia/jargon/master/resources/img/database/Redo%20Log/ib_logfile.jpg)

如图所示：

  - write pos 是当前记录的位置，一边写一边后移，写到最后一个文件末尾后就回到 0 号文件开头
  - checkpoint 是当前要擦除的位置，也是往后推移并且循环的，擦除记录前要把记录更新到数据文件

write pos 和 checkpoint 之间还空着的部分，可以用来记录新的操作。

如果 write pos 追上 checkpoint，表示写满，这时候不能再执行新的更新，得停下来先擦掉一些记录，把 checkpoint 推进一下。

Redo log 文件是循环写入的，在覆盖写之前，总是要保证对应的脏页已经刷到了磁盘。

在InnoDB内部，逻辑上 `ib_logfile` 被当成了一个文件，对应同一个 space id。由于是使用 512 字节 block 对齐写入文件，可以很方便的根据全局维护的 LSN 号计算出要写入到哪一个文件以及对应的偏移量。

Redo log 文件是循环写入的，在覆盖写之前，总是要保证对应的脏页已经刷到了磁盘。在非常大的负载下，Redo log 可能产生的速度非常快，导致频繁的刷脏操作，进而导致性能下降，通常在未做checkpoint 的日志超过文件总大小的76%之后，InnoDB 认为这可能是个不安全的点，会强制的preflush 脏页，导致大量用户线程 stall 住。如果可预期会有这样的场景，我们建议调大redo log文件的大小。可以做一次干净的 shutdown，然后修改 Redo log 配置，重启实例。

### Redo 写盘操作

有几种场景可能会触发redo log写文件：

  1. Redo log buffer空间不足时
  2. 事务提交
  3. 后台线程
  4. 做checkpoint
  5. 实例shutdown时
  6. binlog切换时

我们所熟悉的参数 `innodb_flush_log_at_trx_commit` 作用于事务提交时，这也是最常见的场景：

  - 当设置该值为 1 时，每次事务提交都要做一次 fsync，这是最安全的配置，即使宕机也不会丢失事务；
  - 当设置为 2 时，则在事务提交时只做 write 操作，只保证写到系统的 page cache，因此实例 crash 不会丢失事务，但宕机则可能丢失事务；
  - 当设置为 0 时，事务提交不会触发 redo 写操作，而是留给后台线程每秒一次的刷盘操作，因此实例 crash 将最多丢失 1 秒钟内的事务。

下图表示了不同配置值的持久化程度：

![](https://raw.githubusercontent.com/taojintianxia/jargon/master/resources/img/database/Redo%20Log/innodb_flush_log_at_trx_commit.png)

显然对性能的影响是随着持久化程度的增加而增加的。通常建议在日常场景将该值设置为 1，但在系统高峰期临时修改成 2 以应对大负载。

### Redo checkpoint

InnoDB的redo log采用覆盖循环写的方式，而不是拥有无限的 redo 空间；即使拥有理论上极大的 redo log 空间，为了从崩溃中快速恢复，及时做 checkpoint 也是非常有必要的。

InnoDB的master线程大约每隔 10 秒会做一次 redo checkpoint，但不会去 preflush 脏页来推进 checkpoint 点。

通常普通的低压力负载下，page cleaner 线程的刷脏速度足以保证可作为检查点的 lsn 被及时的推进。但如果系统负载很高时，redo log 推进速度过快，而 page cleaner 来不及刷脏，这时候就会出现用户线程陷入同步刷脏并做同 checkpoint 的境地，这种策略的目的是为了保证 redo log 能够安全的写入文件而不会覆盖最近的检查点。

### 其他

redo log 有一个缓存区 `Innodb_log_buffer`，默认大小为 8M，Innodb 存储引擎先将重做日志写入 innodb_log_buffer 中。

![](https://raw.githubusercontent.com/taojintianxia/jargon/master/resources/img/database/Redo%20Log/innodb_log_buffer.jpg)

然后会通过以下三种方式将 innodb 日志缓冲区的日志刷新到磁盘：

  1. Master Thread 每秒一次执行刷新 Innodb_log_buffer 到重做日志文件
  2. 每个事务提交时会将重做日志刷新到重做日志文件
  3. 当 redo log 缓存可用空间少于一半时，重做日志缓存被刷新到重做日志文件

有了 redo log，InnoDB 就可以保证即使数据库发生异常重启，之前提交的记录都不会丢失，这个能力称为 `crash-safe`。

CrashSafe 能够保证 MySQL 服务器宕机重启后：

  - 所有已经提交的事务的数据仍然存在。
  - 所有没有提交的事务的数据自动回滚。

### 引用
1. 如何恢复到半个月内任意一秒的状态？
2. MySQL 日志系统之 redo log 和 binlog
3. 详细分析MySQL事务日志redo log

