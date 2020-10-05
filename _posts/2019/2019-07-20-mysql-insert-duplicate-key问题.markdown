---
layout: article
key: f5e88829-05ee-4969-acdd-25d83586e276
title: "mysql insert duplicate key问题"
date: 2019-07-20 20:09:11 +0800
categories: [java, mybatis, mysql]
tags: [java, mybatis, mysql]
---

# 问题描述

在使用mybatis且没有用spring时，想实现一个业务逻辑是insert一个对象到
数据表中，同时该对象某些/某个字段是唯一索引，所以想判断如果该记录在数据
表中已经存在则update该记录。    

第一种解决方案：先select查询是否存在该记录，然后在选择insert/update，
这样一是效率不高，再者，存在如果并发产生的线程不安全的问题。

第二种解决方案：先insert该记录，如果数据库中存在冲突，则会成功插入；如果
发生冲突，则会抛出`SQLIntegrityConstraintViolationException`，
然后捕获异常，如果是`SQLIntegrityConstraintViolationException`,
则选择update该记录，如果不是此异常则继续抛出。（？？？这种情况会产生幻读么？）

第三种解决方案(推荐)：`INSERT INTO .. ON DUPLICATE KEY` 使用MYSQL特有的语句。

# 解决办法

因为 SQLIntegrityConstraintViolationException

## 第二种解决方案

下面这个实现，更想说明的另外一个问题，那就是关于异常被包装后的捕获。     
Mybatis如果试图直接捕获`SQLIntegrityConstraintViolationException`编译器
会报错的。因为首先Mybatis官方不推荐捕获异常，其次此框架把异常都给包装起来了。    

一次使用`e.getCause()`来捕获其包装的异常，并进一步处理。

```java
package com.demo.service;

import java.sql.SQLIntegrityConstraintViolationException;

import com.demo.dao.UserDao;
import com.demo.model.User;
import org.apache.ibatis.session.SqlSession;

import static com.demo.util.MyBatisUtil.sqlSessionFactory;

/**
 * @ClassName com.demo.service.UserService
 * @Desciption
 * @Author Shu WJ
 * @DateTime 2019-07-20 17:42
 * @Version 1.0
 **/
public class UserService {

  /** 处理 mybatis 异常
   *  另外考虑使用 on duplicate key 关键词*/
  public void insertSelection(User user) {
    try (SqlSession sqlSession = sqlSessionFactory.openSession()) {
      UserDao userDao = sqlSession.getMapper(UserDao.class);
      try {
        userDao.insert(user);
      } catch (Exception e) {
        if (null == e.getCause()) {
          // 没有被包装的异常，继续抛出
          throw e;
        }

        Throwable nested = e.getCause();
        if (nested instanceof SQLIntegrityConstraintViolationException) {
          // 因违背主键约束/唯一索引/外键所产生的异常
          // 在这里，想实现的逻辑是，如果insert user，发现已经存在该id的记录时，更新该记录
          userDao.updateSelection(user);
        }
      }
      sqlSession.commit();
    }
  }
}
```

## 第三种实现方案

解释：下面如果不存在冲突则直接插入，若表中已有相关记录便执行之后这update语句。

```java
  @Insert({" insert into user(id, name, password) values(#{id}, #{name}, #{password}" +
    " on duplicate key update password=#{password}"})
  int insertOrUpdate(User user);
```

# 额外探索

## 并发insert

Mysql是其默认的隔离等级，可重复读。
User id为自增字段，且name是唯一索引。
下面是建表语句

```SQL
CREATE TABLE `user` (
  `id` bigint(32) NOT NULL AUTO_INCREMENT,
  `name` varchar(255) NOT NULL DEFAULT '',
  `password` varchar(255) NOT NULL DEFAULT '',
  `updated_time` datetime NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
  `created_time` datetime NOT NULL DEFAULT CURRENT_TIMESTAMP,
  PRIMARY KEY (`id`),
  UNIQUE KEY `uq_name` (`name`)
) ENGINE=InnoDB AUTO_INCREMENT=26 DEFAULT CHARSET=utf8
```

### 事务A
```bash
19:12:54 mysql> begin;
Query OK, 0 rows affected (0.00 sec)

19:13:50 mysql> insert into user(id, name, password) values(23, "23", "1");
Query OK, 1 row affected (0.00 sec)

mysql> select * from user;
+----+--------+----------+---------------------+---------------------+
| id | name   | password | updated_time        | created_time        |
+----+--------+----------+---------------------+---------------------+
| 21 | failed | 0        | 2019-07-20 15:01:23 | 2019-07-20 15:01:23 |
| 22 | 22     | 1        | 2019-07-20 19:10:58 | 2019-07-20 19:10:58 |
| 23 | 23     | 1        | 2019-07-20 19:13:50 | 2019-07-20 19:13:50 |
+----+--------+----------+---------------------+---------------------+
3 rows in set (0.00 sec)

19:19:18 mysql> commit;
Query OK, 0 rows affected (0.00 sec)

```

### 事务B
```bash
19:13:29 mysql> begin;
Query OK, 0 rows affected (0.00 sec)

19:13:57 mysql> select * from user;
+----+--------+----------+---------------------+---------------------+
| id | name   | password | updated_time        | created_time        |
+----+--------+----------+---------------------+---------------------+
| 21 | failed | 0        | 2019-07-20 15:01:23 | 2019-07-20 15:01:23 |
| 22 | 22     | 1        | 2019-07-20 19:10:58 | 2019-07-20 19:10:58 |
+----+--------+----------+---------------------+---------------------+
2 rows in set (0.00 sec)

19:14:19 mysql> insert into user(id, name, password) values(23, "23", "1");
ERROR 1205 (HY000): Lock wait timeout exceeded; try restarting transaction
19:14:56 mysql> insert into user(id, name, password) values(23, "23", "1");
ERROR 1062 (23000): Duplicate entry '23' for key 'PRIMARY'
19:19:35 mysql> select * from user;
+----+--------+----------+---------------------+---------------------+
| id | name   | password | updated_time        | created_time        |
+----+--------+----------+---------------------+---------------------+
| 21 | failed | 0        | 2019-07-20 15:01:23 | 2019-07-20 15:01:23 |
| 22 | 22     | 1        | 2019-07-20 19:10:58 | 2019-07-20 19:10:58 |
+----+--------+----------+---------------------+---------------------+
2 rows in set (0.00 sec)

19:19:44 mysql> commit;
Query OK, 0 rows affected (0.00 sec)

19:19:47 mysql> select * from user;
+----+--------+----------+---------------------+---------------------+
| id | name   | password | updated_time        | created_time        |
+----+--------+----------+---------------------+---------------------+
| 21 | failed | 0        | 2019-07-20 15:01:23 | 2019-07-20 15:01:23 |
| 22 | 22     | 1        | 2019-07-20 19:10:58 | 2019-07-20 19:10:58 |
| 23 | 23     | 1        | 2019-07-20 19:13:50 | 2019-07-20 19:13:50 |
+----+--------+----------+---------------------+---------------------+
3 rows in set (0.00 sec)
```

### 分析结果

- 事务A中insert一个记录后, 在没有commit时，事务B中select是看不见这条记录的。

- 事务A中insert一个记录后，在没有commit时，事务B中如果试图insert一条primary/unique
一样的记录时，不会成功`Lock wait timeout exceeded`, 说明该记录被锁住了。这里会等待一段
时间，等待锁被释放，即事务A commit。

- 事务A中insert一个记录后，在commit后，事务B中如果试图insert一条primary/unique
一样的记录时，不会成功，会出现Duplicate entry错误。

- 事务A中insert一个记录后，在commit后，事务B中可以select可以看见该记录。


### 深入分析

具体原因，涉及到MySQL的事务隔离等级，以及其锁机制的相关实现了。这个等待后续，研究相关博客和
官方文档再更新了。

# 参考文档
-