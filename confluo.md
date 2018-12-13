# Confluo: Millisecond-level Queries on Large-scale Live Data

[Press Release](https://rise.cs.berkeley.edu/blog/confluo-millisecond-level-queries-on-large-scale-streaming-data/)
[Research Paper](https://people.eecs.berkeley.edu/~anuragk/papers/confluo.pdf)
[Github](https://ucbrise.github.io/confluo/)

## Introduction
近日，UC Berkeley的RISE实验室开源了多数据流实时分析平台Confluo，其实质是一个时间序列数据库。

根据阅读其使用手册，PR，和研究论文得到的信息，这里作一个简单总结。

## 数据模型
Confluo的数据模型由以下几个核心概念组成：

### Store
类似MYSQL中的数据库，一个Confluo程序可以管理多个stores。

### Atomic MultiLog
这类似MYSQL中的表，一个store可以有多个表。每个Atomic MultiLog都有一个schema。

Schema必须包含一个timestamp域，其它的域必须是定长的（也就是说varchar这种变长数据类型Confluo是不支持的）。下面是一个schema的例子：

```
{
  timestamp: LONG,
  op_latency_ms: DOUBLE,
  cpu_util: DOUBLE,
  mem_avail: DOUBLE,
  lg_msg STRING(100)
}
```

### Filter, Aggregate and Trigger
在一个Atomic MultiLog（以下简称log）上，用户可以定义一个filter，以下是一个filter的例子

```
mlog->add_filter("low_resources", "cpu_util>0.8 || mem_avai<0.1")
```

上述例子在前面定义的log上创建了一个名为low_resources的filter，插入的记录中只要cpu_util大于80%或者mem_avail小于10%即满足该filter的过滤条件。

Filter过后的数据用来干什么呢？在filter上，用户可以定义一个aggregate。以下是一个aggregate的例子

```
mlog->add_aggregate("max_latency_ms", "low_resources", "MAX(op_latency_ms)")
```

上述例子在filter low_resources上定义了一个名为max_latency_ms的aggregate，它使用滑动窗口来计算符合过滤条件的记录的最大操作延迟。

有了aggregate又能起什么用呢？Confluo允许你在aggregate上定义triggers。以下是一个trigger的例子

```
mlog->install_trigger("high_latency_trigger", "max_latency > 1000")
```

上述例子在aggregate max_latency上定义了一个名为high_latency_trigger的trigger。但条件满足时，这个trigger可以触发发送警报都操作。

### Index

为支持高效的查询，Confluo允许用户为log上的attribute创建索引。以下是一个index的例子

```
mlog->add_index("op_latency_ms")
```

## 使用

### 进程模型
Confluo有standalone和embedded两种使用模式。

在Standalone模式下，Confluo作为一个标准的daemon服务器运行，它提供有java/c++/python的客户端通过Apache
Thrift和confluo服务器通信。

在Embededd模式下，Confluo以header-only的形式加入到你C++项目的源码中。你可以直接调用其API。

### 数据插入

以下是一个通过调用Confluo API向前面定义的log插入数据的例子

```
size_t off1 = mlog->append({"100", "0.5", "0.9",  "INFO: Launched 1 tasks"});
size_t off2 = mlog->append({"500", "0.9", "0.05", "WARN: Server {2} down"});
size_t off3 = mlog->append({"1001", "0.9", "0.03", "WARN: Server {2, 4, 5} down"});
```

### 查询

Confluo支持online和offline两种查询方式。

Online查询类似数据库的continous query，只对新到来的数据起作用。Offline查询类似普通的数据库查询，搜索所有的历史数据。

以下是一个查询的例子

```
auto record_stream = mlog_query_filter("low_resources", 0, UINT64_MAX);
for (auto s = record_stream; !s.empty(); s = s.tail()) {
  std::cout << s.head().to_string();
}
```

这个例子定义了一个在filter low_resources上的online查询。第二个和第三个参数定义了被查询数据的时间范围（这里实际上定义了一个无限大的范围）。
这个查询返回一个流，基于这个流用户可以进行filter, map等操作。

## 核心算法与架构

### 假设
系统基于两条重要假设来进行性能优化：
1. 所有记录是append-only，插入后就不会删除和修改
2. 一个log内的所有记录大小是固定的，构成记录的每个域的大小和在记录内的位置也是固定的

### 架构

见下图 ![arch](/images/confluo_arch.PNG)。

插入的数据以round-robin的方式写入到多个ring buffers中。在ring buffer的消费端是多个writers。它们从buffer读取数据，写入到Atomic MultiLog中。

消费者（比如query engine等）从Atomic MultiLog读取数据，进行各种分析与处理。

### Atomic MultiLog

系统的核心数据结构/算法是Atomic MultiLog - 其实质就是多个lock-free的concurrent log并在一起，各司其职（见下图）。下面我们一个个的过一下。
![logs](/images/confluo_log.PNG)

#### Header Log
所有的log都有一个HeaderLog，用于存储原始数据（raw data）。记录在HeaderLog内的offset被系统用作这条记录的唯一标识符。

*注：假设一个没有filter/aggregate/index存在的log。这个Log就只会有一个HeaderLog。在这种情况下，使用多个ring buffers，一个rb有多个线程取数
据然后插入log。这种设计似乎还不如一个Log就只对应一个ring buffer，只有一个线程从这个RB消费数据。*

#### Index Logs
用户可以指定对相关的域作索引，以提高未来查询的性能。每一个需要索引的域会对应一个单独的IndexLog。IndexLog采用perfect k-ary tree（见下图）。
![kary](/images/confluo_kary.PNG)

索引的value是记录的偏移量。

*注：因为数据在内存中不可能无限存储，而IndexLog又没有时间信息，如何保持索引不会无限增大呢？这一点论文没有讨论，估计是通过偏移量，周期性的
把小于某阈值的偏移量从索引中删除*

#### FilterLogs
对用设定的每一个filter，都有一个FilterLog和其对应。FilterLog采用time index，存储的也是符合过滤条件的记录的偏移量。

#### AggregateLog
对每一个aggregate，*每一个writer线程*都通过Thread-local的方式来保存一个local aggregate log。要取得全局的归总值时，系统访问所有local log，
然后把所有local log最近的值再归总，从而得到全局归总值。这就意味着系统只能支持MIN/MAX等遵从交换律和结合律的归总计算。

#### 原子操作
这是整个文章的核心。

一个Multi AtomicLog由多个individual log-free concurrent logs组成。针对每一个individual log的操作都是原子的，但一个线程针对多个individual logs
的操作不一定是原子的。为了保证Multi AtomicLog end-to-end的原子性，文章采用了如下算法：

1. HeaderLog维护有一个writeTail和readTail。
2. 但用户线程向ring buffer写入记录前，它们通过CPU原子指令，得到一个增加的writeTail值作为该记录的offset。
3. Writer线程从RB读取记录后，先把记录根据offset定义的为止写道HeaderLog中，并且更新其它的individual logs。（这时候不会有write-write冲突，因为
每个记录都被在HeaderLog中提前分配了自己的为止。但Record2先于Record1被写入的情况是有可能发生的）。
4. Write完成上述操作后，检查是否自己正在处理的记录的writeTail和当前的readTail只差一个记录的距离，如果是的话，修改readTail使他等于自己的
writeTail。这个操作也是通过CPU的原子指令比如CAS来完成的。

消费线程在消费数据时，先读取readTail的值，然后把从individual logs中读到的offset大于readTail都丢弃不处理。

这样就实现了系统的原子性。

## 点评
PR中号称系统比KAFKA快N倍，似乎不够严谨，毕竟两者的使用场景不同。

从时间序列数据库的角度看，Confluo更多是一个基于内存的TSDB，没有处理persistent的问题。

论文中没有讨论内存分配/管理，这对高性能的TSDB来说是关键的一环。

不是很清楚为什么作者要采用这种多个writer修改单个log的设计。里面的大量设计都是为了解决这多个writer互斥而引发的。由于系统是一个内存数据库，
且log之间（即数据库表之间）不存在join操作，似乎可以采用以下作法：
* 每个CORE一个线程，每个线程有自己的内存池（可采用SLAB）
* 把LOG再这些线程之间均匀分配
* 每个LOG对应一个RING BUFFER

这种设计下，线程之间没有通信需求，线程处理请求时也没有同步需求。这样的设计更简单，性能应该更好。作者是否还有其它考虑，采用了目前的架构，
只能对比才能最后证明了。