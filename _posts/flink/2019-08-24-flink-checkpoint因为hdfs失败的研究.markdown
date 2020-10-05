---
layout: article
key: 9ee3ed2d-d544-4a3b-b0ec-a49a6c241cc6
title: "flink checkpoint因为hdfs失败的研究"
date: 2019-08-24 23:32:42 +0800
categories: [flink, hdfs]
tags: [flink, hdfs]
---


# 问题描述
运行在yarn上的flink job开启了checkpoint，并使用的是hdfs系统，用户保存hdfs数据。在某些时候频繁的出现下面的异常，导致checkpoint失败，进而导致job频繁重启。

主要的原因
```
AsynchronousException{java.lang.Exception: Could not materialize checkpoint 302995 for operator Window(balabalabala) -> Sink: statistic (4/4).}
```
```
java.io.IOException: Unable to close file because the last block does not have enough number of replicas.
```

完整的异常栈：
```
AsynchronousException{java.lang.Exception: Could not materialize checkpoint 302995 for operator Window(balabalabala) -> Sink: statistic (4/4).}
	at org.apache.flink.streaming.runtime.tasks.StreamTask$AsyncCheckpointExceptionHandler.tryHandleCheckpointException(StreamTask.java:1153)
	at org.apache.flink.streaming.runtime.tasks.StreamTask$AsyncCheckpointRunnable.handleExecutionException(StreamTask.java:947)
	at org.apache.flink.streaming.runtime.tasks.StreamTask$AsyncCheckpointRunnable.run(StreamTask.java:884)
	at java.util.concurrent.Executors$RunnableAdapter.call(Executors.java:511)
	at java.util.concurrent.FutureTask.run(FutureTask.java:266)
	at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1149)
	at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:624)
	at java.lang.Thread.run(Thread.java:748)
Caused by: java.lang.Exception: Could not materialize checkpoint 302995 for operator Window(balabala) -> Sink: statistic (4/4).
	at org.apache.flink.streaming.runtime.tasks.StreamTask$AsyncCheckpointRunnable.handleExecutionException(StreamTask.java:942)
	... 6 more
Caused by: java.util.concurrent.ExecutionException: java.io.IOException: Could not flush and close the file system output stream to hdfs://balabala/chk-302995/f4a9687d-455a-41ee-a0a0-6f705bdca8ac in order to obtain the stream state handle
	at java.util.concurrent.FutureTask.report(FutureTask.java:122)
	at java.util.concurrent.FutureTask.get(FutureTask.java:192)
	at org.apache.flink.util.FutureUtil.runIfNotDoneAndGet(FutureUtil.java:53)
	at org.apache.flink.streaming.api.operators.OperatorSnapshotFinalizer.<init>(OperatorSnapshotFinalizer.java:47)
	at org.apache.flink.streaming.runtime.tasks.StreamTask$AsyncCheckpointRunnable.run(StreamTask.java:853)
	... 5 more
Caused by: java.io.IOException: Could not flush and close the file system output stream to hdfs://balabala/chk-302995/f4a9687d-455a-41ee-a0a0-6f705bdca8ac in order to obtain the stream state handle
	at org.apache.flink.runtime.state.filesystem.FsCheckpointStreamFactory$FsCheckpointStateOutputStream.closeAndGetHandle(FsCheckpointStreamFactory.java:326)
	at org.apache.flink.runtime.state.CheckpointStreamWithResultProvider$PrimaryStreamOnly.closeAndFinalizeCheckpointStreamResult(CheckpointStreamWithResultProvider.java:77)
	at org.apache.flink.runtime.state.heap.HeapKeyedStateBackend$HeapSnapshotStrategy$1.callInternal(HeapKeyedStateBackend.java:765)
	at org.apache.flink.runtime.state.heap.HeapKeyedStateBackend$HeapSnapshotStrategy$1.callInternal(HeapKeyedStateBackend.java:724)
	at org.apache.flink.runtime.state.AsyncSnapshotCallable.call(AsyncSnapshotCallable.java:76)
	at java.util.concurrent.FutureTask.run(FutureTask.java:266)
	at org.apache.flink.util.FutureUtil.runIfNotDoneAndGet(FutureUtil.java:50)
	... 7 more
Caused by: java.io.IOException: Unable to close file because the last block does not have enough number of replicas.
	at org.apache.hadoop.hdfs.DFSOutputStream.completeFile(DFSOutputStream.java:2306)
	at org.apache.hadoop.hdfs.DFSOutputStream.closeImpl(DFSOutputStream.java:2267)
	at org.apache.hadoop.hdfs.DFSOutputStream.close(DFSOutputStream.java:2232)
	at org.apache.hadoop.fs.FSDataOutputStream$PositionCache.close(FSDataOutputStream.java:72)
	at org.apache.hadoop.fs.FSDataOutputStream.close(FSDataOutputStream.java:106)
	at org.apache.flink.runtime.fs.hdfs.HadoopDataOutputStream.close(HadoopDataOutputStream.java:52)
	at org.apache.flink.core.fs.ClosingFSDataOutputStream.close(ClosingFSDataOutputStream.java:64)
	at org.apache.flink.runtime.state.filesystem.FsCheckpointStreamFactory$FsCheckpointStateOutputStream.closeAndGetHandle(FsCheckpointStreamFactory.java:312)
	... 13 more
```

# 定位及解决


## 问题定位
一开始怀疑flink使用不对，或者是flink自身的bug，google搜索以及在flink的社区查询一番后未果。

然后，开始怀疑hadoop组件hdfs的问题，在flink的社区发现有人提交了类似的[报告](https://issues.apache.org/jira/browse/HDFS-14690?jql=text%20~%20%22Unable%20to%20close%20file%20because%20the%20last%20block%20does%20not%20have%20enough%20number%20of%20replicas.%22)。
发现谁已解决状态，开心。
>this is not a bug. this is the expected behavior if somehow the NameNode is slow. You may configure a few parameters to work around this problem. E.g. client config dfs.client.block.write.locateFollowingBlock.retries. Default is 5. Setting it to 10 or 15 should help alleviate the problem greatly.     
>Additionally you should also check NameNode to understand why it was slow – GC pauses? heavy I/O? long running RPCs?

大意是说，这个不是bug，而是某些状况导致hdfs namenode变慢进而导致这个问题的，可以考虑配置相关参数等等。
建议查看namenode的GC状态、rpc连接、IO相关的方面。

按照建议查看了namenode节点的TCP连接状态，发现处于TIME_WAIT态的tcp连接数在这段时间的确异常的高。均值约为10w-30w之间。遂停止了flink job，然后第二天等待TIME_WAIT状态的tcp数量下降到均值为1k左右后，重新提交flink job，并未触发该问题。

初步定位问题是namenode的相关异常导致了flink job的异常。



## 问题解决

待与hdfs组件维护开发人员沟通后，后续增加。


# 更多思考

## HDFS的RPC协议
- 使用Protobuf作为传输序列化框架
- 网络传输层描述了Client和Server之间消息的传输方式，Hadoop采用了基于TCP/IP的socket机制。

更多的HDFS RPC实现原理，则需要后续学习。

## TIME_WAIT是什么及其作用
### TIME_WAIT
TIME_WAIT 是这么一种状态：TCP 四次握手结束后，连接双方都不再交换消息，但主动关闭的一方保持这个连接在一段时间内不可用。

### 作用
- 主动关闭的一方再最后回应了ACK报文之后，该ACK报文有丢失的可能，如果丢失，对端会重发FIN报文，因此目的之一是为ACK报文的丢失做应对措施。
- 主动关闭的一方再回应ACK报文之后，如果立刻关闭本连接，后续有可能新的连接复用该连接，这样就会有可能导致原连接的部分报文因为某些原因重新到来，导致与新连接的报文相混淆，扰乱正常报文。

### MSL
- MSL:Maximum Segment Lifetime,报文网络中最大存活时间
> RFC 793 \[Postel 1981c] 指出MSL为2分钟。然而，实现中的常用值是30秒，1分钟，或2分钟

### 时间为何是2MSL
- 保证全双工的 TCP 连接正常终止    
主动关闭的一方回应的ACK报文最大存活MSL，假设在存活时间段的最后也没有到达对方（此为一次MSL），则对方会重发FIN报文，该FIN报文假设在存活的MSL的最后到达了本端（此为一次MSL），则来得及重回ACK，否则就认为对端已正常关闭，那好，本端也可以关闭连接了。

- 保证网络中迷失的数据包正常过期    
有足够的时间让这个连接不会跟后面的连接混在一起（你要知道，有些自做主张的路由器会缓存IP数据包，如果连接被重用了，那么这些延迟收到的包就有可能会跟新连接混在一起）。TCP 下的 IP 层协议是无法保证包传输的先后顺序的。如果双方挥手之后，一个网络四元组（src/dst ip/port）被回收，而此时网络中还有一个迟到的数据包没有被 B 接收，A 应用程序又立刻使用了同样的四元组再创建了一个新的连接后，这个迟到的数据包才到达 B，那么这个数据包就会让 B 以为是 A 刚发过来的。

### 数量
由于 TIME_WAIT 的存在，每个连接被主动关闭后，这个连接就要保留 2MSL（60s） 时长，一个网络四元组也要被冻结 60s。机器默认可被分配的端口号约有 30000 个（可通过 /proc/sys/net/ipv4/ip_local_port_range 文件查看。

## 大量TIME_WAIT带来的危害

- TIME_WAIT过多可能引起无法对外建立新连接

## lunix 查看状态的命令
### shell命令

`netstat -an | awk '/tcp/ {print $6}' | sort | uniq -c`

```bash
SHUSHU:~ shushu$ netstat -an | awk '/tcp/ {print $6}' | sort | uniq -c
   1 CLOSED
  92 ESTABLISHED
   8 FIN_WAIT_2
  19 LISTEN
   6 SYN_SENT
  89 TIME_WAIT
   1 com.apple.network.tcp_ccdebug
```

### 状态含义
- CLOSED：无连接是活动的或正在进行
- LISTEN：服务器在等待进入呼叫
- SYN_RECV：一个连接请求已经到达，等待确认
- SYN_SENT：应用已经开始，打开一个连接
- ESTABLISHED：正常数据传输状态
- FIN_WAIT1：应用说它已经完成
- FIN_WAIT2：另一边已同意释放
- TIMED_WAIT：等待所有分组死掉
- CLOSING：两边同时尝试关闭
- TIME_WAIT：另一边已初始化一个释放
- LAST_ACK： 


# 参考文档
- https://juejin.im/post/5c7e242b51882529df67194a
- https://juejin.im/entry/5c170f506fb9a049ec6afa1e

