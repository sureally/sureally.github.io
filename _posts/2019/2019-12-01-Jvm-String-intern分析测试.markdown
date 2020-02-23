---
layout: article
title: Jvm-String-intern分析测试
date: 2019/12/1 17:11
categories: [java, jvm]
tags: [java, jvm]
---

```java
package com.coding.jdk;

import org.junit.jupiter.api.Assertions;
import org.junit.jupiter.api.Test;

/**
 * @Author shu wj
 * @Date 2019/12/1 16:09
 * @Description 测试java jdk String 相关的方法
 *
 * 前提描述：在jdk 1.6 之后，jvm中的常量池移到了堆中，而不是之前和堆是两个地方。
 * 现在使用的jdk8
 * 1. String str = "stringIntern"; // 这种是直接在常量池新建的对象(如果不存在的话)，返回的是常量池字符串的引用
 * 2. String str =  new String("string"); // 这种是在常量池先新建了string对象，然后在堆区新建了一个字符串对象，并返回引用
 * 3. String str = "string" + "Intern"; // 这种会被优化到，直接在常量池分配的新对象，并返回引用，如果存在就直接返回引用；
 * 4. String str = "Intern"; String str1 = "string" + str; // 这种是将str1当作对象分配的，并不是从常量池拿到的引用，而是堆区。
 *
 * String#Intern 这个方法的作用是，如果常量池有该字符串则返回该引用，如果没有则将**存储一份堆区的引用到常量池中**(只是引用多了一份)。在jdk 1.6 前，是拷贝
 * 一份到常量池。
 *
 * PS: 下面描述中，堆区表示除常量池以外的堆区。
 */
public class StringTest {
// 下面的测试需要一个个运行
  @Test
  public void testString01() {
    String str1 = "string";
    String str2 = new String("string");

    Assertions.assertNotSame(str1, str2);
    Assertions.assertEquals(str1, str2);
  }

  @Test
  public void testString02() {
    String str1 = "string" + "Intern";
    String str2 = "string" + "Intern";

    Assertions.assertSame(str1, str2); // 是相同的引用
    Assertions.assertEquals(str1, str2);
  }

  @Test
  public void testString03() {
    String str = "Intern";
    String str1 = "string" + "Intern"; // 被优化到常量池拿到的字符串的引用
    String str2 = "string" + str; // 这里是在堆区新建的对象并拿大的引用
    String str3 = "stringIntern";

    Assertions.assertNotSame(str1, str2); // 是不同的引用
    Assertions.assertEquals(str1, str2);

    Assertions.assertNotSame(str2, str3);
    Assertions.assertSame(str1, str3);
  }

  @Test
  public void testIntern01() {
    String str1 = "stringIntern";
    String str2 = new String("string") + new String("Intern");
    String str3 = str2.intern(); // 拿到的常量池对象

    Assertions.assertNotSame(str1, str2);
    Assertions.assertEquals(str1, str2);

    Assertions.assertSame(str1, str3);
  }

  @Test
  public void testIntern02() {
    String str2 = new String("string") + new String("Intern"); // 堆区
    String str1 = "stringIntern"; // 因为常量池中不存在stringIntern，所以在常量池新建该对象并返回引用。
    String str3 = str2.intern(); // 因为常量池存在str1，所以返回的是str1这个引用

    Assertions.assertNotSame(str1, str2);
    Assertions.assertEquals(str1, str2);
    Assertions.assertSame(str1, str3);
    Assertions.assertNotSame(str2, str3);
  }

  @Test
  public void testIntern03() {
    String str2 = new String("string") + new String("Intern"); // 堆区
    String str3 = str2.intern();  // 因为，此时，常量池并没有stringIntern这个字符串，所以存一份str2的引用到常量池，并返回。
    String str1 = "stringIntern"; // 这个来自常量池，但是，其实真正的对象是来自堆区

    Assertions.assertSame(str1, str2);
    Assertions.assertEquals(str1, str2);
    Assertions.assertSame(str1, str3);
    Assertions.assertSame(str2, str3);
  }

  @Test
  public void testIntern05() {
    String str2 = new String("string") + new String("Intern"); // 堆区
    str2.intern(); // 调用这个后，会在常量池增加一个str2的引用
    String str1 = "stringIntern"; // 实际拿到的是str2的引用
    Assertions.assertSame(str1, str2);
    Assertions.assertEquals(str1, str2);
  }

  @Test
  public void testIntern06() {
    String str2 = new String("string") + new String("Intern"); // 堆区
    String str1 = "stringIntern"; // 常量池
    Assertions.assertNotSame(str1, str2);
    Assertions.assertEquals(str1, str2);
  }

}

```