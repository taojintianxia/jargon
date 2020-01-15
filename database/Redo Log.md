## MySQL 中有六种日志文件，分别是：
  - 重做日志（redo log）
  - 回滚日志（undo log）
  - 二进制日志（binlog）
  - 错误日志（errorlog）
  - 慢查询日志（slow query log）
  - 一般查询日志（general log）
  - 中继日志（relay log）

其中 `重做日志` 用来保证事务的持久性，`回滚日志` 用来保证事务的原子性以及 InnoDB 的 MVCC。`二进制日志` 也与事务操作有一定的关系。这三种日志，对理解 MySQL 中的事务操作有着重要的意义。

## 重做日志 Redo Log
 
和大多数关系型数据库一样，InnoDB 记录了对数据文件的物理更改，并保证总是日志先行，也就是所谓的 WAL，即在持久化数据文件前，保证之前的 redo log 已经写到磁盘。

LSN(log sequence number) 用于记录日志序号，它是一个不断递增的 unsigned long 类型整数。在 InnoDB 的日志系统中，LSN 无处不在，它既用于表示修改脏页时的日志序号，也用于记录checkpoint，通过 LSN，可以具体的定位到其在 redo log 文件中的位置。

InnoDB的redo log可 以通过参数 `innodb_log_files_in_group` 配置成多个文件，另外一个参数 `innodb_log_file_size` 表示每个文件的大小。因此总的 redo log 大小为：

```
innodb_log_files_in_group * innodb_log_file_size
```
Redo log文件以 `ib_logfile[number]` 命名，日志目录可以通过参数 `innodb_log_group_home_dir` 控制。Redo log 以顺序的方式写入文件文件，写满时则回溯到第一个文件，进行覆盖写。（但在做redo checkpoint 时，也会更新第一个日志文件的头部 checkpoint 标记，所以严格来讲也不算顺序写）。

在InnoDB内部，逻辑上 `ib_logfile` 被当成了一个文件，对应同一个 space id。由于是使用 512 字节 block 对齐写入文件，可以很方便的根据全局维护的 LSN 号计算出要写入到哪一个文件以及对应的偏移量。

