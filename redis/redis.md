## nosql

数据库瓶颈：

- 数据总量太大，放不下
- 索引一个机器的内存放不下
- 访问量大，一个实例搞不定

![image-20200727133445217](C:\Users\dht24\AppData\Roaming\Typora\typora-user-images\image-20200727133445217.png)

**Mysql读写主从分离**

![image-20200727133745580](C:\Users\dht24\AppData\Roaming\Typora\typora-user-images\image-20200727133745580.png)

**分表分库+水平拆分+mysql集群**

![image-20200727134107025](C:\Users\dht24\AppData\Roaming\Typora\typora-user-images\image-20200727134107025.png)



**为什么使用NoSQL（Not Only SQL）?**

传统关系型数据库性能限制。

这一类型的数据存储不需要固定的模式，无需多余操作就可以横向扩展。



**NoSQL特点：**

- 易扩展。数据之间无关系，这样就非常容易扩展。
- 大数据量高性能。
- 多样灵活的数据模型。NoSQL无需事先为要存储的数据
- **KV+Cache+Persistence**



**当下的应用是SQL和NoSQL一起使用**



我们的问题：多数据源多数据类型的存储问题



趋冷的数据：存到关系型数据库

文档数据库：MongoDB



 **聚合模型分类：**

- KV键值对（Redis）
- Bson
- 列族（HBase）
- 图形



**CAP+BASE：**

**CAP只能三选二**

- C:  Consistency（强一致性）
- A：Availability（可用性）
- P：Partition tolerance（分区容错性）



## Redis（Remote Dictionary Server）

**五大数据类型：**

- String
- Hash (Map<String, Object>)
- List
- Set
- zset (sorted set)

