---
layout: article
title: "Spring框架使用注解@Autowired 的警告"
date: 2019-08-25 22:28:08 +0800
categories: Spring
---


# 描述
在使用Spring的注解@Autowired的时候，idea报了一个警告。

```
@Autowired // 这里被警告
private AlarmHostGroupMapper hostGroupMapper;
```

警告内容：
> Field injection is not recommended

即，使用变量依赖注入的方式不被推荐。然后，idea解决策略是
> Always use constructor based dependency injection in your beans. Always use assertions for mandatory dependencies
即，总是使用构造器的方式强制注入。

# 依赖注入

## 三种方式

- 变量(field)注入
``` java
@Autowired
UserDao userDao;
```
- 构造器注入
```java
final UserDao userDao;

@Autowired
public UserServiceImpl(UserDao userDao) {
    this.userDao = userDao;
}
```
- set方法注入

```java
private UserDao userDao;

@Autowired
public void setUserDao(UserDao userDao) {
    this.uerDao = uesrDao;
}
```

## 注入方式比较

### 优点
- 变量方式注入非常简洁，没有任何多余代码，非常有效的提高了java的简洁性。

### 缺点
- 变量方式不能有效的指明依赖。
    > 依赖注入的对象为null，在启动依赖容器时遇到这个问题都是配置的依赖注入少了一个注解什么的，然而这种方式就过于依赖注入容器了，当没有启动整个依赖容器时，这个类就不能运转，在反射时无法提供这个类需要的依赖。 

    > 在使用set方式时，这是一种选择注入，可有可无，即使没有注入这个依赖，那么也不会影响整个类的运行。 

    > 在使用构造器方式时已经显式注明必须强制注入。通过强制指明依赖注入来保证这个类的运行。



# 总结
## 依赖注入的思想

依赖注入的核心思想之一就是被容器管理的类不应该依赖被容器管理的依赖，换成白话来说就是如果这个类使用了依赖注入的类，那么这个类摆脱了这几个依赖必须也能正常运行。然而使用变量注入的方式是不能保证这点的。 
既然使用了依赖注入方式，那么就表明这个类不再对这些依赖负责，这些都由容器管理，那么如何清楚的知道这个类需要哪些依赖呢？它就要使用set方法方式注入或者构造器注入。

## 总结
变量方式注入应该尽量避免，使用set方式注入或者构造器注入，这两种方式的选择就要看这个类是强制依赖的话就用构造器方式，选择依赖的话就用set方法注入。

# 参考文档

- https://blog.csdn.net/zhangjingao/article/details/81094529
