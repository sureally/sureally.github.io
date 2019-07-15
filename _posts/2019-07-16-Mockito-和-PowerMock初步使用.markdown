---
layout: article
title: "Mockito 和 PowerMock初步使用"
date: 2019-07-16 00:11:26 +0800
categories: Java
---


# 前话

在工作之前，Java中的单元测试几乎写得不多。但是，工作之后，经常觉得自己代码写得不佳，因此经常重构，这样凸显单元测试的重要行。同时，在不断写单元测试的过程中，发现**测试驱动开发**是有一定道理的，一则在写测试用例的时候，会让自己更加理解业务逻辑以及需要实现的功能，而来一定程度上减少由于重构或者修复bug时的时间花费。

# Mockito 和 PowerMock 简介

Mockito与PowerMock都是Java流行的一种Mock框架，使用Mock技术能让我们隔离外部依赖以便对我们自己的业务逻辑代码进行单元测试，在编写单元测试时，不需要再进行繁琐的初始化工作，在需要调用某一个接口时，直接模拟一个假方法，并任意指定方法的返回值。

Mockito的工作原理是通过创建依赖对象的proxy，所有的调用先经过proxy对象，proxy对象拦截了所有的请求再根据预设的返回值进行处理。PowerMock则在Mockito原有的基础上做了扩展，通过修改类字节码并使用自定义ClassLoader加载运行的方式来实现mock静态方法、final方法、private方法、系统类的功能。

PowerMock直接依赖于Mockito，所以如果项目中已经导入了PowerMock包就不需要再单独导入Mockito包，如果两者同时导入需要小心PowerMock和Mockito不同版本之间的兼容问题。

PowerMock 可以满足一些Mockito不能满足的功能，比如私有方法、静态方法、系统方法等的mock。

# 使用过程中遇到的问题

## `doAnswer` 与 `doReturn`

###  `doReturn` 

#### 功能
和 `thenReturn` 可以的功能类型。    
大部分blog上推荐 `doReturn` 。

#### 用法
> You should use `thenReturn` or `doReturn` when you know the return value at the time you mock a method call. This defined value is returned when you invoke the mocked method.

当你知道mock的那个方法调用时应该返回什么值的这个时候来使用
`thenReturn` or `doReturn`。

### `doAnswer`
#### 功能

#### 用法
> `doAnswer` is used when you need to do additional actions when a mocked method is invoked, e.g. when you need to compute the return value based on the parameters of this method call.

> Use `doAnswer()` when you want to stub a void method with generic Answer.
Answer specifies an action that is executed and a return value that is returned when you interact with the mock.

简单来说，`doAnswer` 一样可以有返回值，但是通过实现 `Answer` 接口，可以在方法被调用时执行一些额外的逻辑。这样实际使用的过程中，某些情况下是及其方便的。

## `spy` and `mock`
### `mock`
>A mock in mockito is a normal mock in other mocking frameworks (allows you to stub invocations; that is, return specific values out of method calls).    

Mock object replace mocked class entirely, returning recorded or default values.

### `spy` (recomment)
>A spy in mockito is a partial mock in other mocking frameworks (part of the object will be mocked and part will use real method invocations).    

>When spying, you take an existing object and "replace" only some methods. This is useful when you have a huge class and only want to mock certain methods (partial mocking). 

# demo

```java
// TODO: 待练习
```


# 参考文档
- https://juejin.im/post/5be7eaf26fb9a049d4415278
- https://stackoverflow.com/questions/36615330/mockito-doanswer-vs-thenreturn/36627077
- https://stackoverflow.com/questions/28295625/mockito-spy-vs-mock