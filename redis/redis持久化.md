# Redis持久化
redis支持RDB、AOF两种持久化机制，持久化功能有效地避免因线程退出造成的数据丢失问题，当下次重启时利用之前的持久化文件即可实现数据恢复。

## RDB

RDB持久化是把当前线程数据生成快照保存到硬盘的过程。

### 触发机制

- 手动触发
 + <font color=#0099ff>save命令</font>：阻塞当前redis服务器，直到RDB过程完成为止 
 + <font color=#0099ff>bgsave命令</font>：Redis进程执行fork操作创建子进程，RDB由子进程负责，阻塞只发生在fork阶段
- 自动触发
 + 使用save相关配置：
 + 从节点执行全量复制操作，主节点自动执行bgsave
 + 执行debug reload命令触发save
 + 执行shutdown命令时，没有开启AOF则自动执行bgsave

### 优缺点
- 优点
 + RDB是一个紧凑压缩的二进制文件，非常适合备份，全量复制等场景
 + Redis加载RDB恢复数据远远快于AOF的方式
- 缺点
 + 没办法做到实时持久化/妙计持久化。bgsave执行fork创建子进程，属于重量级操作，频繁执行成本很高
 + RDB文件使用特定二进制格式保存，存在老版本无法兼容新版本格式问题

## AOF

以日志的方式记录每次写命令，重启时再重新执行AOF文件中的命令达到恢复数据的目的。解决了数据持久化的实时性，是Redis持久化的主流方式。
### 流程
- 命令写入
  + 写入内容是文本协议格式：兼容性、避免二次处理开销、可读性方便直接修改和处理
  + 先写入AOF缓冲：直接追加到硬盘，性能完全取决于当前硬盘负载。Redis提供多种缓冲区同步硬盘的策略，在性能和安全性方面做出了平衡
- 文件同步
  + AOF缓冲区同步文件策略：
  
   1. <font color=#0099ff>always</font>：命令写入缓存区后调fsync同步到AOF文件
   2. <font color=#0099ff>everysec</font>：命令写入aof_buf后调write操作，write完成后线程返回。fsync同步文件操作由专门线程每秒调用一次
   3. <font color=#0099ff>no</font>：命令写入aof_buf后调write操作，不对AOF文件做fsync同步，同步硬盘由操作系统负责，周期最长30秒
- 文件重写
  + 解决问题：命令不断写入AOF，文件越来越大
  + 操作：把Redis进程内的数据转化为写命令同步到新AOF文件过程
  + 原因：
  
   1. 进程内超时数据不再写入
   2. 旧的AOF文件含有无效命令
   3. 多条写命令可以合并为一个
  
  + 触发
  
   1. 手动触发：bgrewriteaof命令
   2. 自动触发： <font color=#0099ff>auto-aof-rewrite-min-size</font>（运行AOF重写时文件最小体积）和<font color=#0099ff>auto-aof-rewrite-percentage</font>（当前AOF文件空间和上一次重写后AOF文件空间的比值）
- 重启加载
  + 服务器重启时的数据恢复

