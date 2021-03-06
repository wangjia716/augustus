#复制
##一、分类

Redis的复制功能分为同步和命令传播两个操作

###1.同步（全量复制）：

从服务器发送<font color=#0099ff>SLAVEOF</font>命令，要求从服务器复制主服务器时，从服务器首先需要执行同步操作，即将从服务器的数据库状态更新为主服务器的数据库状态（主服务器生成RDB文件并发送给从服务器）

新版<font color=#0099ff>PSYNC</font>命令具有完整重同步和部分重同步两种模式

 - 完整重同步
    
     用于初次复制情况
 - 部分重同步（命令传播）
   
     用于处理断线后复制情况，当从服务器在断线后重新连接主服务器时，如果条件允许，主服务器可以将主从服务器连接断开期间执行的写命令发送给服务器
     
     功能实现由三个部分构成：
     + 复制偏移量：
     
     主从服务器分别维护一个复制偏移量（判断两者是否处于一致状态）
     + 复制积压缓冲区：
     
     当主服务器进行命令传播时，它不仅会将写命令发送给所有从服务器，还会将写命令入队到复制积压缓冲区里面。
     
     当服务器重新连上主服务器时，从服务器会通过<font color=#0099ff>PSYNC</font>命令将自己的复制偏移量<font color=#0099ff>offset</font>发送给主服务器，主服务器会根据这个复制偏移量来决定对从服务器执行何种同步操作。
     
     1) 如果<font color=#0099ff>offset</font>之后的数据还在复制积压缓冲区，执行部分重同步操作。
     
     2) 相反，执行完整重同步操作。 
     + 服务器运行ID：
     服务器断线后并重新连接上一个主服务器时，从服务器将向当前连接的主服务器发送之前保存的运行ID
     
     1) 如果相同，主服务器执行部分重同步操作
     
     2) 如果不相同，主服务器执行完整重同步操作
  
###2.命令传播（部分复制）

主服务器会将自己执行的写命令，也即是造成主从服务器不一致的写命令，发送给从服务器执行

##二、实现步骤
###1.设置主服务器的地址和端口
<font color=#0099ff>SLAVEOF 127.0.0.1 6379</font>
###2.建立套接字连接
从节点内部通过每秒运行的定时任务维护复制相关逻辑，当定时任务发现存在新的主节点后，会尝试与该节点建立网络连接。
###3.发送PING命令
连接建立成功后从节点发送ping请求进行首次通信，目的：

- 检测主从之间的网络套接字是否可用
- 检测主节点在当前是否可接受处理命令

###4.身份验证（权限验证）
如果主节点设置了<font color=#0099ff>requirepass</font>参数，则需要密码验证，从节点必须配置<font color=#0099ff>masterauth</font>参数保证与主节点相同的密码才能通过验证；如果验证失败复制将终止，从节点重新发起复制流程。
###5.发送端口信息
在身份验证步骤之后，从服务器将执行命令<font color=#0099ff>REPLCONF listening-port <port-number></font>, 想主服务器发送从服务器的监听端口号。主服务器收到这个命令后，会将端口号记录在从服务器所对应的客户端状态<font color=#0099ff>slave\_listening\_port</font>属性中
###6.同步
从服务器向主服务器发送PSYNC命令，执行同步操作，并将自己的数据库更新至主服务器数据库当前所处的状态
###7.命令传播
主服务器一直将自己执行的写命令发送给从服务器

##三、心跳检测
在命令传播阶段，从服务器默认会以每秒一次的频率，向主服务器发送命令：

```
REPLCONF ACK <replication_offset>
```
其中，<font color=#0099ff>replication_offset</font>是从服务器当前的复制偏移量

作用： 

- 检测主从服务器的网络连接状态
- 辅助实现<font color=#0099ff>min-slaves</font>选项：防止主服务器在不安全的情况下执行写命令
- 检测命令丢失



##四、常见问题解决思路
###1.读写分离

- 复制数据延迟：编写外部监控程序监听主从节点的复制偏移量，延迟较大时触发报警或者通知客户端避免读取延迟过高的从节点
- 读到过期数据：
  Redis内部维护过期数据删除策略
  + 惰性删除：每次读取时检查键是否超时，超时执行del命令删除键对象
  + 定时删除：定时任务会循环采样一定数量的键，发现超时删除
  Redis在3.2版本解决了这个问题，从节点读取数据之前会检查过期时间来决定是否返回数据
- 从节点故障问题：客户端维护可用从节点列表，当从节点故障时立刻切换到其他从节点或主节点上  


###2.主从配置不一样
###3.规避全量配置

- 第一次建立复制：无法避免。建议在低峰时进行操作或者尽量避免使用大数据量的Redis节点
- 节点运行ID不匹配：架构上避免。提供故障转移功能，手动提升从节点为主节点或者采用支持自动故障转移的哨兵或集群方案
- 复制积压缓冲区不足：根据网络中断时长，写命令数据量分析出合理的积压缓冲区大小

###4.规避复制风暴
 
 大量从节点对同一主节点或者对同一台机器的多个主节点短时间内发起全量复制的过程
 
 - 单主节点复制风暴：减少主节点挂载从节点的数量，或者采用树状复制结构，加入中间层从节点用来保护主节点
 - 单机器复制风暴：单台机器部署多个Redis实例。
  避免方法：
  + 应该把主节点尽量分散在多台机器上
  + 当主节点所在机器故障后提供故障转移机制，避免机器回复后进行密集的全量复制


