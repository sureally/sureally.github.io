---

layout: article
title: 2020-05-03-Guava-EventBus解析学习
date: 2020/5/3 23:44
categories: [Java, Guava]
tags: [Java, Guava]
root-path: ../..
---

# 简介

EventBus是Guava的事件处理机制，是设计模式中的观察者模式（生产/消费者编程模型）的优雅实现。对于事件监听和发布订阅模式，EventBus是一个非常优雅和简单解决方案，我们不用创建复杂的类和接口层次结构。

> It is not a general-purpose publish-subscribe system, nor is it intended for interprocess communication.

# 源码解析

## 结构

1. Eventbus、AsyncEventbus：事件发送器
2. Event：事件承载单元
3. SubscriberRegistry：订阅者注册器，将订阅者注册到event上，即将有注解Subscribe的方法和event绑定起来
4. Dispatcher：事件分发器，将事件的订阅者调用来执行
5. Subscriber、SynchronizedSubscriber：订阅者，并发订阅/同步订阅

## 运行原理

1. Eventbus是基于注册监听的方式来运行的，因此，首先需要先实例化Eventbus，然后才会有事件及监听者。

2. 注册监听者

   底层就是将类eventListener中所有注解有Subcribe的方法与其Event对放在一个map中(一个event可以对应多个Subcribe方法)

3. 事件发送：执行指定事件类型的订阅者（包含了method），从订阅者中获取指定事件的订阅者，然后按照规则（同步、异步）执行指定的方法。（如果一个事件没有订阅者，会被当作“deadEvent”）

4. EventBus和AsyncEventBus的区别

   从字面理解，AsyncEventBus是异步的EventBus，EventBus则是同步的。

   - EventBus中的executor为MoreExecutor.diretExecutor()，其execute方法直接执行线程的run方法，即同步调用run方法。
   - EventBus的dispatcher方法为PerThreadQueueDispatcher

   EventBus整个过程都是同步方法执行的，运行流程如下

   ![image-20200504181426487](/assets/images/guava/image-20200504181426487.png)

   AsyncEventBus的dispatcher为LegacyAsyncDispatcher，executor为自己指定的线程池。运行流程如下（虚线为线程池异步调度）

   ![image-20200504181559396](/assets/images/guava/image-20200504181559396.png)

5. AllowConcurrentEvents的作用

   如果订阅者方法上有注解AllowConcurrentEvents，则返回Subscriber，否则，返回SynchronizedSubscriber。SynchronizedSubscriber的字面意思为同步订阅者。

   即没有使用注解AllowConcurrentEvents的订阅者，在并发环境中，都是串行执行。这在高并发环境中，会严重影响性能。





# 用例

# 注意事项

- 在高并发的环境下使用AsyncEventBus时，发送事件可能会出现异常，因为它使用的线程池，当线程池的线程不够用时，会拒绝接收任务，就会执行线程池的拒绝策略，如果需要关注是否提交事件成功，就需要将线程池的拒绝策略设为抛出异常，并且try-catch来捕获异常

