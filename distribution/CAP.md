# CAP 定理

[CAP](https://en.wikipedia.org/wiki/CAP_theorem) CAP 定理是分布式系统设计中最基础，也是最为关键的理论。它指出，分布式数据存储不可能同时满足以下三个条件：

  - **一致性（Consistency）**：每次读取要么获得最近写入的数据，要么获得一个错误
  - **可用性（Availability）**：每次请求都能获得一个（非错误）响应，但不保证返回的是最新写入的数据。
  - **分区容忍（Partition tolerance）**：尽管任意数量的消息被节点间的网络丢失（或延迟），系统仍继续运行。

