# Kafka Multi-Tenancy - 160 Billion Daily Messages on One Shared Cluster at LINE

[Video](https://www.confluent.io/kafka-summit-sf18/kafka-multi-tenancy)

这是LINE工程师Yuto在2018 Kafka Summit上的一个分享，着重介绍了LINE在Kfaka多租户上的一些实践。本文主要关注他对Kafa一个性能问题的
甄别，分析以及解决的过程。

## 问题

在LINE的生产环境中，99-tile的响应时间一般在20ms左右，但有时能观察到50x-100x的恶化。这个恶化是怎么产生的？

## 数据采集

从下两图可以看出，在响应时间恶化时，对应有不小的磁盘读操作；同时，网络线程的CPU利用率很高。

![Latency](/images/yotu_latency.png)
![diskread](/images/yotu_diskread.png)
![networkthr](/images/yotu_networkthr.png)

## 分析

### Kafka Broker的请求处理算法

下面几个图勾勒了Broker处理一个fetch请求的过程。

![Rh1](/images/yotu_rh1.png)
![Rh2](/images/yotu_rh2.png)
![Rh3](/images/yotu_rh3.png)
![Rh4](/images/yotu_rh4.png)

系统有多个网络线程，它们负责从client socket读取请求，并把这些请求放到一个request queue中。每个网络线程还有一个响应队列，里面存放有
请求的处理结果。网络线程还负责从响应队列取到响应，然后写入client socket。

一个网络线程会负责多个client sockets；这些clients共享一个响应队列。

系统还有一个Request Handler线程池。池中的线程从全局唯一的请求队列读取请求，进行处理，然后把结果放到对应网络线程的响应队列中。

### 假设与求证

在一切正常时（见下图），对一个响应的处理时很快的。网络线程把一些元数据从jvm heap拷贝到socket，然后通过sendfile()调用，以zero-copy的
形式把数据直接从kernel的Page cache拷贝到socket。这个过程大概是100微秒。

![normal1](/images/yotu_normal1.png)
![normal2](/images/yotu_normal2.png)

但如果fetch请求希望读取的topic数据不在page cache呢？这时候，（见下图）sendfile()需要先把数据从disk读到page cache，再从page cache拷贝到
socket。这个过程大概需要50毫秒。

![slow1](/images/yotu_slow1.png)

更糟糕的是，因为多个clients共享一个网络线程，一个慢的head-of-line响应的处理，会增加响应队列中所有响应的处理延迟。

那么什么情况下，fetch请求的数据会不在page cache中？有以下两种情况：
1. 消费者试图读取老数据
2. follower试图从leader恢复老数据
这两种情况在生产实践中应该是比较常见的。

作者通过systemtab，对上述假设作了验证（见下图）

![slow2](/images/yotu_slow2.png)

### 解决方案

网络线程慢是因为其在调用sendfile()时，希望读取的topic数据不在page cache中。如果我们能（大概率）保证sendfile()被调用时topic数据在
page cache中，这个问题就解决了。

怎么做到这一点？作者让request handler线程在把响应放到响应队列之前，调用一次sendfile(/dev/null)。因为/dev/null的写操作实现并不真正
对内存进行任何操作，所以这个sendfile()调用会很快完成。在完成过程当中，request handler线程会被柱塞，但因为request handler线程有
多个，所以不存在head-of-line效应。