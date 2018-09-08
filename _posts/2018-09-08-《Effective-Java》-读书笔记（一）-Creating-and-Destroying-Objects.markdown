---
layout: post
title:  "Creating and Destroying Objects"
date:   2018-09-08 19:59:40 +0800
categories: LeetCode
---

### 1. 静态工厂方法替代构造函数

#### 类获取自己实例的手段  
普通方法：公有的构造函数  
推荐方法：通过类提供一个公有的静态工厂方法，即一个返回类实例的静态方法，注意这和设计模式中的工厂模式不一样。  
下面这行代码，将原始数据类型`boolean`转换为`Boolean`对象的引用。
```java
public static Boolean valueOf(boolean b){
    return b ? Boolean.TRUE : Boolean.FALSE;
}
```

#### 推荐方法的优点与缺点
**优点**  
a. 相比于构造函数，静态工厂方法拥有名字。  
比如：构造函数`BigInteger(int, int, Random)` 返回一个 `BigInteger`，然而一个更好的方法是一个静态工厂方法
`BigInteger.probablePrime` （在JDK 1.4 中加入）。  
在给定的signature下，一个类只能拥有一个构造函数。传统克服此问题的手段是，利用不同的参数列表来重载构造函数，
但是这是一种糟糕的手段，它会带来一些问题，API的使用这将不知道哪个构造函数是哪个，最终可能使用错误；同时，在
阅读此类代码时，读者在不查看参考文档时，不能清晰的明白这些构造函数各自的含义。  
而使用静态工厂方法在合适的命名的情况下，可以克服重载构造函数带来的缺点。  
b. 



