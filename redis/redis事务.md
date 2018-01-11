# Redis事务
Redis通过<font color=#0099ff> MULTI </font>、<font color=#0099ff> EXEC </font>、<font color=#0099ff> WATCH </font>等命令来实现事务功能。事务提供了一种将多个命令请求打包，然后一次性、按顺序地执行多个命令的机制，并且在事务执行期间，服务器不会中断事务而改去执行其他客户端的命令请求，它会将事务中的所有命令都执行完毕，然后才去处理其他客户端的命令请求。



## 1.事务的实现
下面给了一个事务的简单例子：

```
127.0.0.1:6379> multi
OK
127.0.0.1:6379> set "name" "wangjia06"
QUEUED
127.0.0.1:6379> get "name"
QUEUED
127.0.0.1:6379> set "company" "dianping"
QUEUED
127.0.0.1:6379> get "company"
QUEUED
127.0.0.1:6379> set age 28
QUEUED
127.0.0.1:6379> get age
QUEUED
127.0.0.1:6379> exec
1) OK
2) "wangjia06"
3) OK
4) "dianping"
5) OK
6) "28"
```

退出一个事务可以用<font color=#0099ff> MULTI </font>命令, 此时再执行事务会报错：

```
127.0.0.1:6379> multi
OK
127.0.0.1:6379> get "company"
QUEUED
127.0.0.1:6379> get "age"
QUEUED
127.0.0.1:6379> discard
OK
127.0.0.1:6379> exec
(error) ERR EXEC without MULTI
```

一个事务包括三个步骤：

- 事务开始：事务以<font color=#0099ff> MULTI </font>开始，返回OK命令。
- 命令入队：每个事务命令成功进入队列后，返回<font color=#0099ff> QUEUED </font>。
- 事务执行：<font color=#0099ff> EXEC </font>执行事务。

Redis事务内会遇到两种错误：

- 队列入队不成功：语法错误，错误的命令名字或者一些重要的条件如OOM条件。这种客户端会进行校验，如果入队成功则返回<font color=#0099ff> QUEUED </font>，否则返回错误。如果一个命令入队失败，大多数客户端会终止该事务。

```
127.0.0.1:6379> get name
"wangjia08"
127.0.0.1:6379> multi
OK
127.0.0.1:6379> set name "wangjia09"
QUEUED
127.0.0.1:6379> get
(error) ERR wrong number of arguments for 'get' command
127.0.0.1:6379> get name
QUEUED
127.0.0.1:6379> exec
(error) EXECABORT Transaction discarded because of previous errors.
127.0.0.1:6379>
```
- 命令执行时报错：如对一个String值进行列表操作。所有其他的命令会被正常执行即使在事务执行中一些命令失败。

```
127.0.0.1:6379> multi
OK
127.0.0.1:6379> set sex man
QUEUED
127.0.0.1:6379> lpop sex
QUEUED
127.0.0.1:6379> exec
1) OK
2) (error) WRONGTYPE Operation against a key holding the wrong kind of value
127.0.0.1:6379> get sex
"man"
127.0.0.1:6379>
```

Redis不支持事务回滚功能，原因大概有两点：

- 一个redis命令执行失败，一般是语法错误或者应该在开发阶段被检测出来，而不是在生产环境。
- redis追求简单、速度快，所以不提供事务回滚功能.


## 2.WATCH命令的实现
<font color=#0099ff> watch </font>命令给redis事务提供了一个<font color=#0099ff> CAS </font>功能。它是一个乐观锁，检查被监视的健是否至少有一个已经被修改过了，如果是的话拒绝执行，并返回nil代表执行失败。

如下面例子，在一个客户端开启事务前对键“name”进行监视，若在这个事务执行前，另一个客户端修改了键“name”的值，之后事务执行后就会报nil。

```
127.0.0.1:6379> watch "name"
OK
127.0.0.1:6379> multi
OK
127.0.0.1:6379> set age 10
QUEUED
127.0.0.1:6379> get "name"
QUEUED
127.0.0.1:6379> exec
(nil)
```
```
127.0.0.1:6379> set "name" "wangjia08"
OK
```

<font color=#0099ff> watch </font>命令监视多个键：

```
127.0.0.1:6379> watch "name" "age"
OK
127.0.0.1:6379> multi
OK
127.0.0.1:6379> get "name"
QUEUED
127.0.0.1:6379> get "age"
QUEUED
127.0.0.1:6379> exec
(nil)
127.0.0.1:6379>
```
```
127.0.0.1:6379> set "age" 11
OK
```

## 3.事务的ACID性质

- 原子性：redis事务多个操作当做一个整体来执行，要么执行事务中的所有操作，要么一个也不执行。
- 一致性：“一致”指的是数据符合数据库本身的定义和要求，没有包含非法的或无效的错误数据
 + 入队错误：服务器拒绝执行该事务。
 + 执行错误：出错的命令会被服务器识别出来，并进行相应的错误处理，不会对数据库做任何修改，也不会对数据一致性造成影响。
 + 服务器停机：<font color=#0099ff> RDB </font>、<font color=#0099ff> AOF </font>还原数据库状态
- 隔离性：单线程执行事务，且服务器保证事务期间不会对事务进行中断
- 持久性：当一个事务执行完毕时，执行这个事务所得的结果已经被保存到永久性存储介质（比如硬盘）里面了。Redis事务的持久性由Redis所使用的持久化模式决定（无持久化模式、<font color=#0099ff> RDB </font>、<font color=#0099ff> AOF </font>）。Redis工作在无持久化模式下时，事务无持久性。在<font color=#0099ff> AOF </font>模式下<font color=#0099ff> appenfsync </font>为<font color=#0099ff> no </font>时，事务也无持久性
