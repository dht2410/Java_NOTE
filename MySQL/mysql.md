![image-20200426180419940](C:\Users\dht24\AppData\Roaming\Typora\typora-user-images\image-20200426180419940.png)

一个数据库里可包含多个数据表，数据存在表里。

SQL: Structured Query Language

##### 数据库表

- 类——数据表
- 属性——表中字段
- 对象——记录

**Linux下启动：service mysql start**

**Windows下**

启动mysql：以管理员身份运行，net start mysql

登录MySQL：mysql -u root -p      (-u后面跟用户，-p后面跟密码)



数据定义语言：简称DDL(Data Definition Language)，用来定义数据库对象：数据库，表，列等。关键字：create，alter，drop等 
数据操作语言：简称DML(Data Manipulation Language)，用来对数据库中表的记录进行更新。关键字：insert，delete，update等
数据控制语言：简称DCL(Data Control Language)，用来定义数据库的访问权限和安全级别，及创建用户。
数据查询语言：简称DQL(Data Query Language)，用来查询数据库中表的记录。关键字：select，from，where等



SQL通用语法

不区分大小写

数据类型(常用)：int     double	varchar	date



**数据库操作 database**

create database 数据库名；	创建数据库

show databases;	查看数据库

drop database 数据库名；	删除数据库

use 数据库名；	进入该库



创建表：

create table 表名（

​		列名1	数据类型	约束，

​		列名2	数据类型	约束，

​		列名3	数据类型	约束

）；



show tables;	查看表
desc 表名；	查看表的具体结构
drop table 表名；	删除表
show columns from 表名;	展示所有字段信息



主键约束：

保证列的数据的唯一性和非空性

```mysql
可以自增长
create table users(
	uid INT PRIMARY KEY AUTO_INCREMENT, 
    --可变字符，非空
    name VARCHAR(20) NOT NULL,
    age INT
);
```



添加列：

```mysql
ALTER TABLE users ADD tel INT;
```

修改列：

```mysql
ALTER TABLE users modify tel VARCHAR(20);
```

修改列名：

```mysql
ALTER TABLE users change tel telephone DOUBLE;
```

删除列：

```MySQL
ALTER TABLE users drop telephone;
```

修改表名：

```MySQL
rename table users to myusers;
```



向数据表中添加数据insert
insert into 表名（列名1，列名2，列名3）values (值1，值2，值3)；
可以不考虑主键；

```mysql
insert into users (uid,name,age) values (1,"dang",24);
```

也可以

```mysql
insert into users values (1,"dang",24);
```

批量添加
insert into 表名（列名1，列名2，列名3）values (值1，值2，值3)， (值1，值2，值3)；



对数据进行更新操作
update  表名  set  列1=值1，列2=值2  where  条件

```mysql
update users set name="zhuyixuan",age=3 where id=1;
```

修改条件写法：

id<>6	id不等于6
与	and
或	or
非	not
包含	id in {1,2,3,4,5}

```mysql
update users set name="zhuyixuan",age=3 where id=1 AND id=2;
```



删除表中的数据：

delete from 表名 where 条件；

```mysql
delete from users where uid=1;
```



查询：

查询指定列的数据
select 列名1， 列名2 from 表名

```mysql
select uid from users;
```

查询所有列的数据
select * from 表名

查询去掉重复的记录
select distinct 列名 from 表名

查询数据中，直接进行数字计算
select name,age+10 from users;

**条件查询**

**select 字段 from 表名 where 条件**

```mysql
select * from users where age>5;
-- 区间查
select * from users where age between 2 and 30;
-- 展示uid和name字段
select uid,name from users where age between 2 and 30;
-- 或
select * from users where age in (3,22);
-- not
select * from users where name is not null;
-- like 模糊查询 通配符
select * from users where name like 'zhu%';
-- like 几个字符几个下划线
select * from users where name like '_____';
```

**查询，对结果集进行排序**
order by 列名 [desc] [asc]     默认升序

```mysql
select * from users order by age asc;  -- 升序
select * from where age>5 order by age desc;  -- 先过滤再排序
```

**聚合函数**

```MySQL
-- count求和
select count(*) from users;
select count(age) from users;
-- 求和
select sum(age) from users;
select sum(age) as 'sumage' from users where age>5;
-- max min avg
select max(age) from users where age>5;
```

**分组查询**

group by 被分组的列名
必须跟聚合函数

```MySQL
-- age一样的
select sum(age),name from users group by age;
```

