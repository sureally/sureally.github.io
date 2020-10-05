# [blog](https://sureally.github.io/)
语言集中于Java。
框架工具集中于
- Flink
- Springboot
- Kafka
- Mybatis-Mysql

本地测试命令
```shell script
bundle exec jekyll serve
```
当执行 `bundle install` 卡住可以执行 `bundle config mirror.https://rubygems.org https://gems.ruby-china.com`

每篇文章的模板
在这里，指定root-path后，可以使得md的图片使用相对路径，同时服务器中也可以使用图片路径。
```shell script
---
layout: article
title: $TITLE$
date: $DATE$ $TIME$
categories: 
tags: 
root-path: ../..
---
```