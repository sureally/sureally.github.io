---
layout: post
title:  "记录一个Flink sink 到tsdb的一个问题"
date:   2019-07-07 14:14:34 +0800
categories: Flink
---

问题表现
===
- kafka topic 正在被消费且为offset为当前值
- 在前一天启动的时候能够写入到tsdb
- 在中间某个不确定时间点不能写入（查看kafka的确没有输入值）
- 没有反压出现
- 重新启动，又能够写入到tsdb
- flink job 中显示输入到sink算子中有数据在不断进入。

问题定位
===
flink中实现的sink到tsdb的程序。

```scala
/** tsdb
  * by shu wenjun
  * */
class CustomTsdbSink extends SinkFunction[String] {
  private val LOGGER = LoggerFactory.getLogger(classOf[CustomTsdbSink])

  override def invoke(value: String, context: SinkFunction.Context[_]): Unit = {

  val builder = Point.measurement("test-database")

  val ns = System.nanoTime()
  // 提高为ns精度，防止time+tag一样使得influxdb被冲刷掉;
  // 这里也存在小概率可能导致被覆盖的问题
  val timestampNS = TimeUnit.NANOSECONDS.convert(new Date().getTime, TimeUnit.MILLISECONDS)

  // 1000000L ns - ms 之间相隔的单位
  builder.time(timestampNS + ns % 1000000L, TimeUnit.NANOSECONDS)

  builder.addField("value", value)

  val point = builder.build()

  CustomTsdbSink.influxDB.setDatabase("test-measurement")
  CustomTsdbSink.influxDB.write(point)
}

object CustomTsdbSink {
  val influxDB: InfluxDB = InfluxDBFactory.connect(InfluxDBConfig.masters) // 的分布式tsdb
  influxDB.setRetentionPolicy("rp_4w") // 保留策略为4周
  influxDB.enableBatch(BatchOptions.DEFAULTS) // 批量写入
}
```
上述代码使用的批量写入数据的方法，遗留的问题便是，如果在批量写入的数据中存在某条数据超出保留策略时间的话，不仅会导致本次批量写入的数据不能完全写入数据库，而且之后的所有数据都会被阻塞不能发送到数据库。并且在代码中无法捕获异常，而在数据库日志中存在warn信息
`lvl=warn "node 7 failed to write points: partial write: points beyond retention policy dropped=1`

另外，如果注释`influxDB.enableBatch(BatchOptions.DEFAULTS) // 批量写入`这句话，在程序运行中如果出现超出保留策略的数据仅会影响该条消息的写入，并在本地可捕获到相应的异常。





