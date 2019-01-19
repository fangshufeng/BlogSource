---
layout: post
title: "mysql常见操作"
date: 2018-03-11 23:03:16 +0800
comments: true
categories: 
tag:
    其他
---


# 说明：本文旨在记录常见数据库操作

![logo.png](https://upload-images.jianshu.io/upload_images/3279997-766e27779ef40d2e.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)



## 数据库相关

1. 增加数据库;
    `create database 数据库的名字 charset=字符编码;`
    
    ```
    create database testdb charset=utf8;
    ```


2. 查询数据库

    `show databases;`

    ```
    mysql> show databases;
    +--------------------+
    | Database           |
    +--------------------+
    | information_schema |
    | mock_db            |
    | mysql              |
    | performance_schema |
    | sys                |
    | testdb             |
    +--------------------+
    6 rows in set (0.01 sec)
    
    ```
3. 选择数据库
    `use testdb;`
    
    ```
    mysql> use testdb;
    Database changed
    mysql>
    ```
4. 查看正在使用的数据库

    `select database();`

    ```
    mysql> select database();
    +------------+
    | database() |
    +------------+
    | testdb     |
    +------------+
    1 row in set (0.00 sec)
    
    mysql>
    ```
5. 查看使用的数据库的所有表

    `show tables;`
    
    ```
    mysql> show tables;
    Empty set (0.00 sec)
    
    mysql>
    ```
    
6. 删除数据库

    `drop database 数据库名称;`
    
    ```
    mysql> drop database testdb;
    Query OK, 0 rows affected (0.00 sec)
    
    mysql>
    ```
   
7. 数据库备份

    这里用到的是终端的一个重定向`>`的命令
    
   `Mac`下数据库的数据保存的路径`/usr/local/mysql/data`
   
   <!--more-->
   
   进入上面目录
   
    ```
     sudo cd  /usr/local/mysql/data
    
    ```
   
   使用`mysqldump`命令将数据备份到桌面下`~/Desktop/kkk.sql`
   
    ```
    ➜  ~ mysqldump -u root -p testdb > ~/Desktop/kkk.sql;
    Enter password:
    ```
    数据密码就可以在桌面看到kkk.sql了
    
    我们先把旧的`testdb`删除掉
    
    ```
    drop database testdb;
    ```
    
    查看一下
    
    ```
    mysql> show databases;
    +--------------------+
    | Database           |
    +--------------------+
    | information_schema |
    | mock_db            |
    | mysql              |
    | performance_schema |
    | sys                |
    +--------------------+
    5 rows in set (0.00 sec)
    
    mysql>
    ```
    
    我们来创建一个空的名为`testdb`的数据库
    
    ```
    create database testdb2 charset=utf8;
    ```
    
    我们查看下`testdb`
    
    
    ```
    mysql> show databases;
    +--------------------+
    | Database           |
    +--------------------+
    | information_schema |
    | mock_db            |
    | mysql              |
    | performance_schema |
    | sys                |
    | testdb             |
    +--------------------+
    6 rows in set (0.00 sec)
    
    
    mysql> use testdb;
    Database changed
    
    mysql> show tables;
    Empty set (0.00 sec)
    
    ```
    
    恢复数据
    
    
    ```
    ➜  ~ mysql -u root -p testdb < ~/Desktop/kkk.sql;
    Enter password:
    ```
    
    再查一次
    
    ```
    mysql> show tables;
    +------------------+
    | Tables_in_testdb |
    +------------------+
    | person           |
    +------------------+
    1 row in set (0.00 sec)
    
    mysql> select *from person;
    +----+----------------+------+
    | id | name           | city |
    +----+----------------+------+
    |  1 | zhangsanfeng   |      |
    |  3 | zhaomin        |      |
    |  4 | xiaozhao       |      |
    |  5 | jinnmaoshiwang |      |
    |  6 | baimei         |      |
    +----+----------------+------+
    5 rows in set (0.00 sec)
    
    mysql>
    ```
    
## 表相关

1. 增加表

    `create table 表名(列及类型);`
    
    ```
    mysql> create table person( 
         > id int auto_increment primary key,
         > name varchar(20) not null);
    Query OK, 0 rows affected (0.02 sec)
    ```
    
    通过`show tables`查看
    
    ```
    mysql> show tables;
    +------------------+
    | Tables_in_testdb |
    +------------------+
    | person           |
    +------------------+
    1 row in set (0.00 sec)
    
    mysql>
    
    ```
2. 查看表属性信息

    `desc 表名称;`
    
    ```
    mysql> desc person;
    +-------+-------------+------+-----+---------+----------------+
    | Field | Type        | Null | Key | Default | Extra          |
    +-------+-------------+------+-----+---------+----------------+
    | id    | int(11)     | NO   | PRI | NULL    | auto_increment |
    | name  | varchar(20) | NO   |     | NULL    |                |
    +-------+-------------+------+-----+---------+----------------+
    2 rows in set (0.00 sec)
    
    mysql>
    ```
3. 删除表

    `drop table 表名;`
    
    现在有2张表
    
    ```
    mysql> show tables;
    +------------------+
    | Tables_in_testdb |
    +------------------+
    | person           |
    | test2            |
    +------------------+
    2 rows in set (0.00 sec)
    ```
    
    删除`test2`表
    
    ```
    mysql> drop table test2;
    Query OK, 0 rows affected (0.00 sec)
    
    mysql> show tables;
    +------------------+
    | Tables_in_testdb |
    +------------------+
    | person           |
    +------------------+
    1 row in set (0.00 sec)
    ```
        
4. 修改表        

    `alter table 表名 add | change | drop 列名  类型;`
    
    比如：
    
    将`person`表的gender 改成gender2;
    
    ```
    alter table person change gender gender2 bit;
    ```
    
    给person增加`city`列
    
    ```
    alter table person add city varchar(20);
    ```
    
    删除person的gender2
    
    ```
    alter table person drop gender2;
    ```
    
    
## 数据相关

1. 查询表数据

    `select * from 表名;`
    
2. 添加数据

    (1). 全列增加单个： `insert into 表名 values(...);`
    
   ```
   mysql> insert into person values(0,'zhangsanfeng');
   Query OK, 1 row affected (0.01 sec)
   mysql>
   
   ```
   
   查询结果
   
   ```
   mysql> select * from person;
   +----+--------------+
   | id | name         |
   +----+--------------+
   |  1 | zhangsanfeng |
   +----+--------------+
   1 row in set (0.00 sec)
   
   mysql>
   ```
    (2). 全列增加多个： `insert into 表名 values(...),(...);`
    
    ```
    mysql> insert into person values(0,'zhangwuji'),(0,'zhaomin');
    Query OK, 2 rows affected (0.00 sec)
    Records: 2  Duplicates: 0  Warnings: 0
    ```
    
    查看一下

    ```
    mysql> select * from person;
    +----+--------------+
    | id | name         |
    +----+--------------+
    |  1 | zhangsanfeng |
    |  2 | zhangwuji    |
    |  3 | zhaomin      |
    +----+--------------+
    3 rows in set (0.00 sec)
    
    mysql>
    ```

    (3). 单列增加单个： `insert into 表名(列名) values(...);`

    ```
    mysql> insert into person(name) values('xiaozhao');
    Query OK, 1 row affected (0.01 sec)
    ```
    
    查看
    
    ```
    mysql> select * from person;
    +----+--------------+
    | id | name         |
    +----+--------------+
    |  1 | zhangsanfeng |
    |  2 | zhangwuji    |
    |  3 | zhaomin      |
    |  4 | xiaozhao     |
    +----+--------------+
    4 rows in set (0.00 sec)
    
    mysql>
    ```

    (4). 单列增加多个： `insert into 表名(列名) values(...),values(...);`
    
    ```
    mysql> insert into person(name) values('jinnmaoshiwang'),('zhangcuishan');
    Query OK, 2 rows affected (0.01 sec)
    Records: 2  Duplicates: 0  Warnings: 0
    ```
    
    查看
    
    
    ```
    mysql> select * from person;
    +----+----------------+
    | id | name           |
    +----+----------------+
    |  1 | zhangsanfeng   |
    |  2 | zhangwuji      |
    |  3 | zhaomin        |
    |  4 | xiaozhao       |
    |  5 | jinnmaoshiwang |
    |  6 | zhangcuishan   |
    +----+----------------+
    6 rows in set (0.00 sec)
    
    mysql>
    ```

3. 更新数据

    `update 表名 set 列名=值 where 条件`
    
    ```
    mysql> update person set name='baimei' where id=6;
    Query OK, 1 row affected (0.01 sec)
    Rows matched: 1  Changed: 1  Warnings: 0
    ```
    
    查看
    
    ```
    mysql> select *from person;
    +----+----------------+
    | id | name           |
    +----+----------------+
    |  1 | zhangsanfeng   |
    |  2 | zhangwuji      |
    |  3 | zhaomin        |
    |  4 | xiaozhao       |
    |  5 | jinnmaoshiwang |
    |  6 | baimei         |
    +----+----------------+
    6 rows in set (0.00 sec)
    
    mysql>
    ```

4. 删除数据

    `delete from 表名 where 条件`
    
    在person表中删除id为2的数据
    
    ```
    mysql> delete from person where id=2;
    Query OK, 1 row affected (0.01 sec)
    ```
    
    查看
    
    ```
    mysql> select *from person;
    +----+----------------+
    | id | name           |
    +----+----------------+
    |  1 | zhangsanfeng   |
    |  3 | zhaomin        |
    |  4 | xiaozhao       |
    |  5 | jinnmaoshiwang |
    |  6 | baimei         |
    +----+----------------+
    5 rows in set (0.00 sec)
    
    mysql>
    ```


## 常见数据查询操作

### 1. 模糊查询

* like
* %表示任意多个，也即是可有可无
* _表示一个任意字符，至少一个


`person`表内容

```
mysql> select * from person;
+----+----------------+----------+
| id | name           | city     |
+----+----------------+----------+
|  1 | zhangsanfeng   | Shanghai |
|  3 | zhaomin        | 北京      |
|  4 | xiaozhao       |          |
|  5 | jinnmaoshiwang | 北京     |
|  6 | baimei         |          |
+----+----------------+----------+
5 rows in set (0.00 sec)

mysql>
```

查询城市是北京的人的名字

```
mysql> select name from person where city  like '北京';
+----------------+
| name           |
+----------------+
| zhaomin        |
| jinnmaoshiwang |
+----------------+
2 rows in set (0.01 sec)

mysql>
```

查询以`北`开头的人的名字，使用`%`和`_`都可以

```
mysql> select name from person where city  like '北%';
+----------------+
| name           |
+----------------+
| zhaomin        |
| jinnmaoshiwang |
+----------------+
2 rows in set (0.00 sec)


mysql> select name from person where city  like '北_';
+----------------+
| name           |
+----------------+
| zhaomin        |
| jinnmaoshiwang |
+----------------+
2 rows in set (0.00 sec)
```


修改一下`zhaomin`的city区分一下`%`和`_`;


```
mysql> update person set city='北' where id=3;
Query OK, 1 row affected (0.01 sec)
Rows matched: 1  Changed: 1  Warnings: 0

mysql> select * from person;
+----+----------------+----------+
| id | name           | city     |
+----+----------------+----------+
|  1 | zhangsanfeng   | Shanghai |
|  3 | zhaomin        | 北       |
|  4 | xiaozhao       |          |
|  5 | jinnmaoshiwang | 北京     |
|  6 | baimei         |          |
+----+----------------+----------+
5 rows in set (0.00 sec)

```

再运行下上面的

```
mysql> select name from person where city  like '北%';
+----------------+
| name           |
+----------------+
| zhaomin        |
| jinnmaoshiwang |
+----------------+
2 rows in set (0.00 sec)

mysql>
mysql> select name from person where city  like '北_';
+----------------+
| name           |
+----------------+
| jinnmaoshiwang |
+----------------+
1 row in set (0.00 sec)
```

### 2. 聚合查询

| 简写 | 含义 |
| --- | --- |
| min | 最小值 |
| max | 最大值 |
| avg | 平均值 |
| sum | 求和 |
| count | 数量 |

```
mysql> select *from person;
+----+----------------+--------+
| id | name           | city   |
+----+----------------+--------+
|  1 | zhangsanfeng   | 上海   |
|  3 | zhaomin        | 北     |
|  4 | xiaozhao       | 上海   |
|  5 | jinnmaoshiwang | 北京   |
|  6 | baimei         | 上海   |
+----+----------------+--------+
5 rows in set (0.00 sec)
```

求city是上海的人的总和 

```
mysql> select count(*) from person where city='上海';
+----------+
| count(*) |
+----------+
|        3 |
+----------+
1 row in set (0.01 sec)

```

求city是上海的人中id最大的的人

```
mysql> select max(id) from person where city='上海';
+---------+
| max(id) |
+---------+
|       6 |
+---------+
1 row in set (0.01 sec)
```

其他的和上面的类似


### 3. 分组查询

查询每个city的人的总数

```
mysql> select city ,count(*) from person group by city;
+--------+----------+
| city   | count(*) |
+--------+----------+
| 上海   |        3 |
| 北     |        1 |
| 北京   |        1 |
+--------+----------+

```

对结果进行排序,使用`order by`

将上面的结果按照总数进行，升序排序

```
mysql> select city ,count(*) as 总数 from person group by city order by 总数 asc;
+--------+--------+
| city   | 总数   |
+--------+--------+
| 北     |      1 |
| 北京   |      1 |
| 上海   |      3 |
+--------+--------+
3 rows in set (0.00 sec)

```
上面的`as`是取别名



单独看上海的有多少人，使用`having`

```
mysql> select city ,count(*) from person group by city having city='上海';
+--------+----------+
| city   | count(*) |
+--------+----------+
| 上海   |        3 |
+--------+----------+
1 row in set (0.00 sec)
```

分页查询，使用`limit `

`select * from 表名 limit start,count`

`start`，表示从第几个元素开始；
`count`，查询几个


```
mysql> select * from person;
+----+----------------+--------+
| id | name           | city   |
+----+----------------+--------+
|  1 | zhangsanfeng   | 上海   |
|  3 | zhaomin        | 北     |
|  4 | xiaozhao       | 上海   |
|  5 | jinnmaoshiwang | 北京   |
|  6 | baimei         | 上海   |
|  7 | zhangsan1      | 海南   |
|  8 | zhangsan2      | 海南   |
|  9 | zhangsan3      | 海南   |
| 10 | zhangsan4      | 海南   |
| 11 | zhangsan5      | 海南   |
| 12 | zhangsan6      | 海南   |
| 13 | zhangsan7      | 海南   |
| 14 | zhangsan8      | 海南   |
+----+----------------+--------+
13 rows in set (0.00 sec)

```

需求：分页查找，每页2个，从0开始；


```
第一页
mysql> select * from person limit 0,3;
+----+--------------+--------+
| id | name         | city   |
+----+--------------+--------+
|  1 | zhangsanfeng | 上海   |
|  3 | zhaomin      | 北     |
|  4 | xiaozhao     | 上海   |
+----+--------------+--------+
3 rows in set (0.00 sec)

第二页
mysql> select * from person limit 3,3;
+----+----------------+--------+
| id | name           | city   |
+----+----------------+--------+
|  5 | jinnmaoshiwang | 北京   |
|  6 | baimei         | 上海   |
|  7 | zhangsan1      | 海南   |
+----+----------------+--------+
3 rows in set (0.00 sec)

第三页
mysql> select * from person limit 6,3;
+----+-----------+--------+
| id | name      | city   |
+----+-----------+--------+
|  8 | zhangsan2 | 海南   |
|  9 | zhangsan3 | 海南   |
| 10 | zhangsan4 | 海南   |
+----+-----------+--------+
3 rows in set (0.00 sec)

```

`having`和`where`区别

`where`是对原始数据进行筛选；
`having `是对筛选的结果进行筛选。


该文会持续补充。。。

