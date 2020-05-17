---
layout: article
title: 2020-05-12-SpringBoot事物:事物的传播
date: 2020/5/12 22:04
categories: [Spring]
tags: [Spring]
root-path: ../..
---

# 事务的传播是什么？

任何应用都涉及到许多服务或组件，这些服务或组件需要调用其他服务或组件。

事物传播指示任何组件或服务是否会参与或将参加事物，以及如果调用组件/服务已经创建或尚未创建事物，它将如何执行。

> 数据库事务是访问并可能修改数据库内容的单个逻辑工作单元。

# Spring事务的原理

## Spring事务基本原理

Spring事务的本质是数据库对事务的支持，没有数据库的支持，Spring是无法提供事务功能的。对于纯JDBC操作数据库，想用到事务需要的步骤如下

```
// 1. 获取连接
Connection conn = DriverManager.getConnection()
// 2. 开始事务
conn.setAutoCommit(true/false)
// 3. 执行CRUD

// 4. 提交事务/回滚事务
conn.commit() / conn.rollback()
// 5. 关闭连接
conn.close()
```

使用Spring的事务管理功能后，就可以不再写步骤2和步骤4的功能了，而是由Spring自动完成。

那么Spring是如何在我们书写的 CRUD 之前和之后开启事务和关闭事务的呢？解决这个问题，也就可以从整体上理解Spring的事务管理实现原理了

下面简单地介绍下，注解方式为例子:

1. 配置文件开启注解驱动，在相关类和方法上通过注解`@Transaction`标识
2. Spring在启动的时候会解析生成相关的bean，这时候会查看拥有相关注解的类和方法，并且为这些类和方法生成代理，并根据`@Transaction`的相关参数进行相关配置注入，这样就在代理中为我们把相关的事务处理掉了（开启正常提交事务，异常回滚事务）
3. 真正的数据库层的事务提交和回滚是通过 binlog 或者 redo log实现的

## Spring的事务机制

所有的数据访问技术都有事务处理机制，这些技术提供API用来开启事务、提交事务来完成数据操作，或者在发送错误的时候回滚事务。

Spring的事务机制是用统一的机制来处理不同数据访问技术的事务处理。Spring的事务机制提供了一个 `PlatformTransactionManager` 接口，不同的数据访问技术的事务使用不同的接口实现，如表所示。

| 数据访问技术 | 实现                           |
| ------------ | ------------------------------ |
| JDBC         | `DataSourceTransactionManager` |
| JPA          | `JpaTransactionManager`        |
| Hibernate    | `HibernateTransactionManager`  |
| JDO          | `JdoTransactionManager`        |
| 分布式事务   | `JtaTransactionManager`        |

在程序中，定义事务管理器的代码如下

```java
@Bean
public PlatformTransactionManager transactionManager() {
	JpaTransactionManager transactionManager = new JpaTransactionManager();
	transactionManager.setDataSource(datasource());
	return transactionManager;
}
```

## 声明式事务

Spring支持声明式事务，即使用注解来选择需要使用事务的方法，它使用`@Transactional`注解在方法上表明该方法需要事务支持。这是一个基于AOP的实现操作。

```
@Transactional
public void saveSomething(Long  id, String name) {
    //数据库操作
}
```

在此处需要特别注意的是，此`@Transactional`注解来自`org.springframework.transaction.annotation`包，而不是`javax.transaction`。

## AOP代理的两种实现

- Java动态代理，私有方法不会存在接口里，所以就不会被拦截到
- Cglib代理，cglib是子类，私有方法同样不会出现在子类里，也不能被拦截

关于代理的详细分析可以参考 [另一篇Blog]([https://sureally.github.io/java/2019/10/06/%E9%9D%99%E6%80%81%E4%BB%A3%E7%90%86%E5%92%8C%E5%8A%A8%E6%80%81%E4%BB%A3%E7%90%86.html](https://sureally.github.io/java/2019/10/06/静态代理和动态代理.html)).

## Spring事务源码解析

Spring事务怎么挂起的？传播等级怎么实现的？保存点、内嵌事务怎么做回滚和提交？

### Spring事务执行纵览

### TransactionStatus源码解析

### TransactionStatus+TransactionSynchronizationManager的使用

### 提交与回滚解析





# Spring事务的传播属性

所谓spring事务的传播属性，就是定义在存在多个事务同时存在的时候，spring应该如何处理这些事务的行为。这些属性在TransactionDefinition中定义，具体常量的解释见下表

| 常量名称                  | 常量解释                                                     |
| ------------------------- | ------------------------------------------------------------ |
| PROPAGATION_REQUIRED      | Spring默认事务的传播。支持当前事务，如果当前没有事务，就新建一个事务。如果内层服务的异常被外层调用捕获吞掉后，是无法触发回滚的。 |
| PROPAGATION_REQUIRES_NEW  | 新建事务，如果当前存在事务，把当前事务挂起。新建的事务和被挂起是事务没有任何关系，是两个独立的事务，外层事务失败回滚之后，不能回滚内层事务执行的结果；内层事务失败抛出异常，外层事务捕获，也可以不处理回滚操作。 |
| PROPAGATION_SUPPORTS      | 支持当前事务，如果当前没有事务，则以非事务方式执行。         |
| PROPAGATION_MANDATORY     | 支持当前事务，如果当前没有事务，则抛出异常                   |
| PROPAGATION_NOT_SUPPORTED | 以非事务方式执行操作，如果当前存在事务，就把当前事务挂起     |
| PROPAGATION_NEVER         | 以非事务方式执行，如果当前存在事务，则抛出异常               |
| PROPAGATION_NESTED        | 如果一个活动的事务存在，则运行在一个嵌套的事务（外部事务的子事务）中，如果没有活动的事务，则按REQUIRED属性执行。使用了一个单独的事务，这个事务拥有多个可以回滚的保存点。内部事务的回滚不会对外部事务造成影响。它只会对`DataSourceTransactionManager`事务管理器起效。 |

最容易弄混淆的其实是 PROPAGATION_REQUIRES_NEW 和 PROPAGATION_NESTED, 那么这两种方式又有何区别

> PROPAGATION_REQUIRES_NEW 启动一个新的, 不依赖于环境的 "内部" 事务. 这个事务将被完全 commited 或 rolled back 而不依赖于外部事务, 它拥有自己的隔离范围, 自己的锁, 等等. 当内部事务开始执行时, 外部事务将被挂起, 内务事务结束时, 外部事务将继续执行
>
> 另一方面, PROPAGATION_NESTED 开始一个 "嵌套的" 事务, 它是已经存在事务的一个真正的子事务. 潜套事务开始执行时, 它将取得一个 savepoint. 如果这个嵌套事务失败, 我们将回滚到此 savepoint. 潜套事务是外部事务的一部分, 只有外部事务结束后它才会被提交.
>
>   由此可见, PROPAGATION_REQUIRES_NEW 和 PROPAGATION_NESTED 的最大区别在于, PROPAGATION_REQUIRES_NEW 完全是一个新的事务, 而 PROPAGATION_NESTED 则是外部事务的子事务, 如果外部事务 commit, 嵌套事务也会被 commit, 这个规则同样适用于 roll back.



# 数据库隔离等级

| 隔离等级         | 隔离等级值 | 导致的问题                                                   |
| ---------------- | ---------- | ------------------------------------------------------------ |
| Read-Uncommitted | 0          | 脏读                                                         |
| Read-Committed   | 1          | 避免脏读，允许不可重复读/幻读                                |
| Repreatable-Read | 2          | 避免脏读，不可重复读，允许幻读                               |
| Serializable     | 3          | 串行化读，事务只能一个个执行，避免脏读/不可重复读/幻读。执行效率低 |

> 脏读：一个事务对数据进行了修改，但未提交，另一个事务可以读取到未提交的数据。如果第一个事务这个时候回滚，那么第二个事务就读到了脏数据。
>
> 不可重复读：一个事务中发生了两次读数据，第一次读操作和第二次读操作之间，另外一个事务对数据进行了修改，这时候两次读取的数据是不一致的。
>
> 幻读：第一个事务对一定范围的数据进行了批量的修改，第二个事务在这个范围增加一条数据，这时候第一个事务就会丢失对新增数据的修改。

隔离的等级越高越能保证数据的一致性和完整性，但是对并发性能的影响也越大

大多数的数据库默认隔离等级为 Read Commited，比如 SqlServer/Oracle

少数数据库默认隔离级别为：Repeatable Read，比如 Mysql InnoDB

# Spring中的隔离等级

| 常量                      | 解释                                                         |
| ------------------------- | ------------------------------------------------------------ |
| ISOLATION_DEFAULT         | 这是个 `PlatfromTransactionManager` 默认的隔离级别，使用数据库默认的事务隔离级别。另外四个与 JDBC 的隔离级别相对应。 |
| ISOLATION_READ_UNCOMMITED | 这是事务最低的隔离级别，它充许另外一个事务可以看到这个事务未提交的数据。这种隔离级别会产生脏读，不可重复读和幻像读。 |
| ISOLATION_READ_COMMITEAD  | 保证一个事务修改的数据提交后才能被另外一个事务读取。另外一个事务不能读取该事务未提交的数据。 |
| ISOLATION_REPEATABLE_READ | 这种事务隔离级别可以防止脏读，不可重复读。但是可能出现幻像读。 |
| ISOLATION_SERIALIZABLE    | 这是花费最高代价但是最可靠的事务隔离级别。事务被处理为顺序执行。 |

# Springboot 对事务的支持

## 只读事务

> 概念：从这一点设置的时间点开始（时间点a）到这个事务结束的过程中，其他事务所提交的数据，该事务将看不见！（查询中不会出现别人在时间点a之后提交的数据）。

`@Transcational(readOnly=true)` 这个注解一般会写在业务类上，或者其方法上，用来对其添加事务控制。当括号中添加`readOnly=true`, 则会告诉底层数据源，这个是一个只读事务，对于JDBC而言，只读事务会有一定的速度优化。

而这样写的话，事务控制的其他配置则采用默认值，事务的隔离级别(isolation) 为DEFAULT,也就是跟随底层数据源的隔离级别，事务的传播行为(propagation)则是REQUIRED，所以还是会有事务存在，一旦在代码中抛出RuntimeException，依然会导致事务回滚。

应用场合：

1. 如果你一次执行单条查询语句，则没有必要启用事务支持，数据库默认支持SQL执行期间的读一致性；
2. 如果你一次执行多条查询语句，例如统计查询，报表查询，在这种场景下，多条查询SQL必须保证整体的读一致性，否则，在前条SQL查询之后，后条SQL查询之前，数据被其他用户改变，则该次整体的统计查询将会出现读数据不一致的状态，此时，应该启用事务支持。

【注意是一次执行多次查询来统计某些信息，这时为了保证数据整体的一致性，要用只读事务】

# Spring Data JPA对事务的支持

默认情况下，Spring Data JPA 实现的方法都是使用事务的。针对查询类型的方法，其等价于 `@Transactional(readOnly=true)`；增删改类型的方法，等价于 `@Transactional`。可以看出，除了将查询的方法设为只读事务外，其他事务属性均采用默认值。

如果用户觉得有必要，可以在接口方法上使用 `@Transactional` 显式指定事务属性，该值覆盖 Spring Data JPA 提供的默认值。同时，开发者也可以在业务层方法上使用` @Transactional `指定事务属性，这主要针对一个业务层方法多次调用持久层方法的情况。持久层的事务会根据设置的事务传播行为来决定是挂起业务层事务还是加入业务层的事务。

# Mybatis 和 Spring 的事务管理权力之争

// TODO

# 注意事项

## @Transactional注解使用的注意事项

- 正确设置progagation属性

- 正确设置rollbackFor属性

- 只能应用于 public 方法才有效

- 避免 Spring 的AOP的自调用问题

  > 在 Spring 的AOP代理下，只有目标方法由外部调用，目标方法才由 Spring 生成的代理对象来管理，这会造成自调用的问题。
  >
  > 若同一类中的其他没有@Transactional 注解的方法内部调用有@Transactional 注解的方法，有@Transactional 注解的方法的事务被忽略，不会发生回滚。



# 参考

1. [一文带你深入理解 Spring 事务原理](https://mp.weixin.qq.com/s?__biz=MzU5NTgzMDYyMA==&mid=2247487862&idx=2&sn=628b97468d56e9488cfa3877412c96f1&chksm=fe6aa345c91d2a53c1a743277593a6bdccd2debbefbd8669b3613a56b38df5be6c40815c0a33&mpshare=1&scene=1&srcid=0511w3xL9TL21pD35JwGGS09&sharer_sharetime=1589198203540&sharer_shareid=65022ec60cc63b548026c0fc9296ef1a&exportkey=AajazGRzRFvVYwgK4ODZ5G0=&pass_ticket=FjIysWfUYyV9kA3g2Ry8Z7oDXvLAYrmkN//dsusNhfmeGOymMtx8klZN8eyRqKoi#rd)