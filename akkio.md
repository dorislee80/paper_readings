# Sharding the Shards: Managing Datastore Locality at Scale with Akkio
https://www.usenix.org/system/files/osdi18-annamalai.pdf

## Problem
在多个数据中心之间复制数据有两个好处 - 1) 增加数据的availability；2) 能从离请求节点近的数据中心访问数据，从而缩短访问时延。一般情况下，业界认为3复制
是保证数据availability和成本之间较好的平衡点。

FB在全球有数以十记的DC，如何选择其中的三个作为数据复制点？在什么粒度上进行数据复制？数据复制功能如何在应用程序以及底层存储系统之间分割？这是本论文想
回答的问题。

## Solution
### Micro-shard
和其他系统动辄以GB为单位的复制单元不同，akkio的复制单位被称为micro-shard。一个micro-shard的大小从1kb到1mb不等。且micro-shard的内容由应用程序决定，
因为应用程序能更好的知道哪些数据会被一起访问。

### Akkio
FB没有把选择复制点的任务交给应用程序或者下层存储系统（比如Cassandra）。它引入一个中间件AKKIO来完成这个任务。

AKKIO由多个SERVICES，以及client library组成。

Client library - 这个LIBRARY和应用程序连接在一起。应用程序通过LIBRARY的API来实现数据访问，统计信息更新等功能。

定位服务 - 这个服务存放一个MicroShardId -> 底层存储系统ID的映射。当应用程序要访问数据时，它向定位服务发送一个包含有MicroShardId的请求，定位服务返回
它底层存储系统的ID。应用程序用返回的ID调用底层存储系统的client library读取数据。

统计服务 - 应用程序对micro-shard的每一次访问，都会通过一个异步的API汇报到统计服务，用来在计算新的数据复制点时提供依据。

Placement服务 - 应用程序如果感受到micro-shard的访问性能不理想，它会通知Placement服务对这个micro-shard进行复制点的重新计算。Placement服务本身也负责
指挥调度复制过程。

### 调度算法
Placement服务使用以下算法来选择新的数据复制点：

#### 创建新的micro-shard
选择发出创建请求的client所在的datacenter作为primary replica，选择两个负荷较轻的datacenter作为secondary replica。

#### 重新计算数据复制点
先为每个DC计算一个score。这个score等于过去X天从该DC访问这个shard的次数，越近发生的访问，在计算score时，权重越高。
根据这个score，对DC排序，然后排除掉一些有特殊情况的DC。



