# Linux tracing systems and how they fit together

[link](https://jvns.ca/blog/2017/07/05/linux-tracing-systems/)

Linux下有令人眼花缭乱的多个tracing工具，这篇文章把这些工具梳理了一遍。

## 分类方法

作者把这些工具分为以下三类：
* data sources - where the tracing data comes from
* collecting mechanisms - how to collect data from the data sources
* tracing frontends - the tool you actually interact with to analyse data.

![big_picture](/images/tracing_big_picture.png)

## Data sources

![data_sources](/images/tracing_data_sources.png)

上图总结了Linux常见的tracing数据源。这些数据源主要覆盖系统调用（system calls），内核函数调用(Linux kernel function calls），应用
程序函数调用(userspace function calls），以及一些自定义事件。

根据其工作原理的不同，数据源又可以细分为两类：
* probes - 系统修改程序指令来实施tracing，比如kprobe和uprobe。
* tracepoints - 这是编译进程序的代码，可以被enable/disable，比如dtrace probes, lttng-ust, kernel tracepoint。

### kprobes

先看下lwn的介绍：

> KProbes are a debugging mechanism for the Linux Kernel which can also be used for monitoring events inside a production system.
> You can use it to weed out performance bottlenecks, log specific events, trace problems etc.

[slides](https://www.cs.dartmouth.edu/sergey/cs258/2016/kprobes-2016.pdf)

