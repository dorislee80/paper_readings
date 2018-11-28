# One Day, One DataHub, 100 Billion Message: Kafka at LINE

这是LINE的工程师YUTO在KAFKA SUMMIT上的两次演讲，分别介绍了他甄别，改进KAFKA几个性能瓶颈的历程。

[Video 1](https://kafka-summit.org/sessions/single-data-hub-services-feed-100-billion-messages-per-day/)


# KAFKA-4614

Yuto在生产环境中偶尔会观察到这样的现象：系统会产生小数据量的磁盘读，且伴随着读操作，broker处理读请求的延迟会变大。由于消费者总是
读一个partition的last segment。而根据kafka的设计，last segment会被缓存在page cache中，因此这种情形应该不会发生。

为找到造成这个现象的终极原因，Yuto作了以下分析：

1. 使用监控工具观察到现象 
![Performance](/images/yotu_metrics.png)

2. 使用systemtap分析BIO_READ请求的来源
![systemtap](/images/yuto_systemtap.png)