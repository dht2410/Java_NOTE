## 体系结构

数据库是文件集合。数据库实例是程序，用于真实操作数据库。

MySQL表现为**单进程多线程**。

![img](https://ss0.bdstatic.com/70cFvHSh_Q1YnxGkpoWK1HF6hhy/it/u=194710674,2116926636&fm=26&gp=0.jpg)

- 连接池组件
- 管理服务和工具组件
- SQL接口组件
- 查询分析器组件
- 优化器组件
- 缓冲组件
- **插件式存储引擎（核心）**
- 物理文件

连接MySQL：连接MySQL操作是一个连接进程和MySQL数据库实例进行通信。



## InnoDB存储引擎

![image-20200607182832512](C:\Users\dht24\AppData\Roaming\Typora\typora-user-images\image-20200607182832512.png)

后台线程：

- Master Thread
- IO Thread
- Purge Thread
- Page Cleaner Thread

内存：

- 缓冲池。磁盘读取的页放在缓冲池，即一块内存。更新时，先更新页，以一定频率刷新到磁盘。缓冲池包含索引页、数据页、插入缓冲、自适应哈希索引、锁信息、数据字典信息。
- 利用LRU策略更新缓存池。最新读到的页放到LRU列表的5/8处。
- 重做日志(redo log)缓冲。Master Thread每一秒将重做日志缓冲刷新到重做日志文件中。

**Checkpoint技术**

Write Ahead Log策略：当事务提交时，先写重做日志，再修改页。满足事务D的要求。

当数据库宕机时，不需要重做所有的日志。Checkpoint之前的页都已经刷新回磁盘。故数据库只需对Checkpoint后重做的日志进行恢复。

**InnoDB关键特性**

- 插入缓冲（Insert buffer）？。如果非聚集索引页在缓冲池，则直接插入；不在，先放到一个insert buffer对象中。这需要索引是辅助索引，索引不是唯一的。
- 两次写？
- 自适应哈希索引（AHI）。对某些热点页建立哈希索引。O(1)复杂度，快于B+树。
- 异步IO。进行一次IO，不用等到这次IO结束才能继续接下来的操作。
- 刷新邻接页。当刷新一个脏页时，会检测该页所在的区（extent）。



## 文件

- 参数文件
- 日志文件
- socket文件
- pid文件
- MySQL表结构文件。用来存放MySQL表结构定义文件。
- 存储引擎文件

### 参数文件

参数分为静态参数和动态参数。动态参数可以在实例运行中修改。

### 日志文件

包括错误日志、二进制文件、慢查询日志、查询日志

###### （1）错误日志

文件为主机名.err

###### （2）慢查询日志

记录执行时间超过某阈值的sql语句

###### （3）查询日志

记录了所有对MySQL数据库请求的信息。文件为主机名.log

###### （4）二进制文件

文件格式为二进制。记录采用sql语句的方式。逻辑日志。

三种模式：

1. statement：记录每条修改的具体sql，记录量少，但使用某些特定函数时会出现问题。
2. row level：记录哪条数据被怎样修改，记录量大。
3. mixed：两者结合，先用statement，不行的话用row level。

后缀为二进制日志序列号。.index为二进制的索引文件，用来存储产生的二进制日志序号。二进制日志需要手动开启。

默认二进制日志文件最大为1G，超过后会开新的日志文件。

所有未提交事务的二进制日志会被记录到缓存中，默认大小为32k。事务提交时，写入二进制文件。当32k不都用，则要先写入一个临时文件，需要落盘。

- 恢复：
- 复制：通过复制和执行二进制日志使master和slave同步
- 审计：审计有没有注入攻击

### PID文件

MySQL启动实例时，会将自己的进程ID写入pid文件中。主机名.pid

### 表结构定义文件

每个表都有对应的文件。不论采用何种存储引擎，都有一个以frm为后缀名的文件。这个文件记录了该表的表结构定义。

show global variables like "%datadir%"

###  InnoDB存储引擎文件

###### （1）表空间文件

在默认配置下会有一个初始大小为10MB，名为ibdatal的文件。默认表空间文件。所有表的数据存放于此。

设置参数innodb_file_per_table为ON，则每个表有单独的ibd文件。表名.ibd。该文件仅存储该表的数据、索引和插入缓冲BITMAP等信息。

默认为96Kb

###### （2）重做日志文件（redo log）

InnoDB的存储引擎下有两个名为ib_logfile0和ib_logfile1的文件。

每个InnoDB存储引擎至少有1个重做日志文件组，每个文件组下至少有两个重做日志文件。每个重做日志文件大小一致，并以循环写入的方式运行。

与二进制日志的不同：

- 二进制日志会记录所有与MySQL数据库有关的日志。而InnoDB的存储日志只记录有关该存储引擎本身的事务日志。
- 二进制日志记录一个事务的具体操作，逻辑日志。redo log记录每个页更改的物理情况。
- 写入时间不同。二进制日志文件仅在事务提交前进行提交。而事务进行的过程中，不断有重做日志条目被写入重做日志。
- redo log基于crash recovery，binlog基于point-in-time recovery

![image-20200608214553166](C:\Users\dht24\AppData\Roaming\Typora\typora-user-images\image-20200608214553166.png)

写入重做日志文件不是直接写，而是先写入一个重做日志缓冲。然后以一定顺序写入日志文件。

为保证ACID中的持久性。每当有事务提交时，就必须确保事务都已经写入重做日志文件。
