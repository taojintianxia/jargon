# MySQL中有六种日志文件，分别是：
  - 重做日志（redo log）
  - 回滚日志（undo log）
  - 二进制日志（binlog）
  - 错误日志（errorlog）
  - 慢查询日志（slow query log）
  - 一般查询日志（general log）
  - 中继日志（relay log）

其中 `重做日志` 和 `回滚日志` 与事务操作息息相关，`二进制日志` 也与事务操作有一定的关系，这三种日志，对理解 MySQL 中的事务操作有着重要的意义。

## 重做日志 Redo log
 
和大多数关系型数据库一样，InnoDB 记录了对数据文件的物理更改，并保证总是日志先行，也就是所谓的 WAL，即在持久化数据文件前，保证之前的 redo log 已经写到磁盘。