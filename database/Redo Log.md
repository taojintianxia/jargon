# MySQL中有六种日志文件，分别是：
  - 重做日志（redo log）
  - 回滚日志（undo log）
  - 二进制日志（binlog）
  - 错误日志（errorlog）
  - 慢查询日志（slow query log）
  - 一般查询日志（general log）
  - 中继日志（relay log）

其中重做日志和回滚日志与事务操作息息相关，二进制日志也与事务操作有一定的关系，这三种日志，对理解MySQL中的事务操作有着重要的意义。

## 重做日志 Redo log
 
 redo log 是一种基于磁盘的数据结构，用于在崩溃恢复期间，纠正由不完整的事务产生的数据。在普通操作中，redo logo 会将更改数据的请求进行编码，这些请求都是由 SQL statements 或者 低级别的 API 调用的。