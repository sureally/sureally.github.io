---
layout: article
title:  "Flink kafka comsuer 动态感知分区以及producer自定义partitioner"
date:   2019-07-07 13:59:34 +0800
categories: Flink
---

# consumer 动态感知分区

## 场景

flink source kafka时，希望能够动态感知kafka分区的变化，以便及时发现。在实际场景的一个举例是，在日志实时分析系统通知，flink消费的topic里面是实时采集到的某系统的日志信息。一开始是根据系统正常情况下的日志量大小来设计的topic 分区，但是当系统工作不正常时，可能会造成比正常大一个数量级及以上的日志，进一步会导致topic 每个分区的入口流量都很大，然后flink 仅消费原来的分区个数便力不能及了，此时可以通过增加分区个数，让flink consuemr提高其消费上限。 

另外，flink consumer在实例一定的情况下其消费能力是有限的。通过增加消费分区，只是提高flink 的消费速度，遇到上限的情况，只能增加consumer实例来解决。

官方文档
---
> The Flink Kafka Consumer supports discovering dynamically created Kafka partitions, and consumes them with exactly-once guarantees. All partitions discovered after the initial retrieval of partition metadata (i.e., when the job starts running) will be consumed from the earliest possible offset.  
By default, partition discovery is disabled. To enable it, set a non-negative value for flink.partition-discovery.interval-millis in the provided properties config, representing the discovery interval in milliseconds.

表明flink kafka consumer是支持动态发现分区的，通过配置属性`flink.partition-discovery.interval-millis`。


另外：flink kafka consumer 还能动态发现topic。


## producer 未见能动态感知分区

如果在kafka中调整增加了分区，flink 官方文档没有说明能够动态发现分区。

# producer重写partitioner

## 应用描述

flink kafka producer sink 默认的分区策略是`FlinkKafkaPartitioner`，一个sink并行度与一个分区一一对应。比如，sink 并行度为2（分别是
A、B），而分区有四个（分别是a、b、c、d），那么相对应的方式便是，A sink到a，B sink到b。上述默认方式，导致的一个问题就是不能分区数据倾斜问题。实际应用举例，还是一个日志处理系统，存在一个kafka topic作为中间层，上承接若干系统的数据清洗模块，下承接单个规则引擎模块。如此，便希望前置的各个数据清洗模块能够将数据均匀的发送的各个分区，所以需要自定义分区接口。（还有一个办法便是，在flink中sink的并行度设定为和分区个数一样，但是这个方法造成的一个问题便是，资源的浪费）

## 解决方案

- 实现自定义分区    

```
/**
  * 根据record 的hashcode 进行 轮换分区
  * PS: hashcode计算有可能为负整数！
  */
class CustomPartitioner[T] extends FlinkKafkaPartitioner[T] {
  override def partition(record: T, key: Array[Byte], value: Array[Byte], targetTopic: String, partitions: Array[Int]): Int = {
    partitions(math.abs(record.toString.hashCode) % partitions.length)
  }
}
```

## 初始化producer     

```
object CustomKafkaProducer {

  /**
    * 获取flink kafka topic。以round-robin的方式发送给topic的分区
    * @param topic sink kafka topic
    * @return
    */
  def getKafkaProducer(topic: String): FlinkKafkaProducer[String] = {
    val partitioner: FlinkKafkaPartitioner[String] = new CustomPartitioner[String]
    val producer = new FlinkKafkaProducer(
      topic,
      new SimpleStringSchema,
      InitKafkaProducerProperty.properties,
      Optional.of(partitioner))

    producer.setWriteTimestampToKafka(true)
    producer
  }
}
```

这里存在一个坑，如果不是单独语句声明`val partitioner: FlinkKafkaPartitioner[String]`是不能通过编译的。

```
def getKafkaProducer(topic: String): FlinkKafkaProducer[String] = {
    val producer = new FlinkKafkaProducer(
      topic,
      new SimpleStringSchema,
      InitKafkaProducerProperty.properties,
      Optional.of(new CustomPartitioner[String])) // 不能通过编译。

    producer.setWriteTimestampToKafka(true)
    producer
  }
```
