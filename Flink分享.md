#Flink分享

------

[TOC]

## 大数据处理平台

### 大数据发展

最早的数据存储都是以SQL为主，在上个世纪SQL类数据库的理论和应用实践都飞速发展。但是随着数据量的增大、数据纬度（字段）的爆炸式发展，传统的SQL不能跟上日益增长的数据需求，所以大约在05-09年之间，Google分享了**GFS**、**MapReduce**、**Chubby**和**BigTable**的相关论文，为大数据技术奠定了基础。

> - **GFS**就是一个分布式的文件存储，我们常用的**HDFS**可以认为是**GFS**的一个开源实现
> - **MapReduce**是非常著名的分布式计算模型，是后序各种大数据计算模型和框架的基础
> - **Chubby**是一个分布式锁，**Zookeeper**是它的开源实现
> - **BigTable**是一个非关系型数据库，**HBase**是他的一个开源实现

### 流处理和批处理

####批处理

早期的**MapReduce**是一个批处理，批处理就是从文件系统中读取一个数据集，对数据集进行拆分，中间结果储存之后重新分配，再进行拆分、汇聚最后输出结果。

批处理的特点是，可以一次性处理海量的数据，但是实时性不够。因此基于批处理有一个Lamda架构（批处理层、服务层、速度层）

> 批处理层就是通过计算原始数据计算一个视图，服务层就是聚合各个视图展示一些历史报表，速度层就是最新的数据没有办法及时得到跑批的结果，因此最新的数据直接内存中计算，当需要展示最新的数据中，多半是从内存中捞取。
>
> Lamda架构是大数据系统构建中非常经典的一个架构。

Lamda架构的优点和缺点：

1. **数据不变性**：由于历史数据是不会变更的，如果计算过程中有计算错误，只需要很简单的删掉对应的视图，重新跑就可以，尽可能降低了人为影响带来不可逆转的后果。
2. **可拓展**：如果需要计算更多的视图，可以简单的堆机器来解决。
3. **实时性**：由于结合了速度层，所以实时性也有保障。

####流处理

由于批处理的速度层，其实已经是一个流处理的过程，因为永远都有新数据输入，速度层也需要永远输出最近一定时间内的结果，那么为什么不把整个流处理架构提升到整个架构而去维护一个速度层和批处理层，因为这样你需要维护两套逻辑，一套是批处理逻辑，一套是实时逻辑。

基于这样的思想，流处理是解决一个永远没有停止的输入和一个永远没有停止的输出这样一类问题。

典型的架构就是Kappa架构，以Kafka消息作为输入，不同的job消费这些消息，并且输出不同的视图，最后基于这些视图产出结果。

**这两个数据处理方式都有各自擅长的领域，比如流处理适合分析日志、实时告警等，批处理擅长离线计算类，比如历史数据聚合。批处理更多用于处理带有“界限”概念的数据集（非常明确的开始时间、和非常明确的结束时间），流处理更多用于处理实时连续的数据集（没有结束时间）。**

## Flink

Flink同时支持了流处理和批处理，之前的框架都是仅仅支持流或者批，但是Flink提供了一套高度抽象的API，不论你的数据集是流还是批，都是一个处理方式（Flink内部都是用流的方式来处理的，批处理就是一个有界限的流）。

### Flink基本概念

####DataSource

DataSource定义了数据输入，可以是一个数据流，也可以是一个批数据，比如DataSource可以是读取文件（批），也可以是不间断的消费消息队列的消息（流）

#### Transformation

数据处理，针对你读取到的数据进行各种形式的处理

#### Sink

数据落地，分析、清洗之后得到的汇总数据需要落库等操作，Sink就是最终结果的一个储存（可以存数据库、再发一个消息、写入一个文件等）

####并行处理

作为一个大数据的框架肯定需要支持分布式计算，数据流会被进行分区，每个Operator会针对这部分数据执行一个或多个操作，这些操作都是并行处理的。

####窗口

比如统计最近五分钟的交易量，滚动窗口、滑动窗口等。

####Time

Flink有三个时间的概念

- **事件时间**

  这个时间就是一个操作真正的发生时间

- **摄取时间**

  这个时间就是一个操作被作为DataSource被Flink集群获取的时间

- **处理时间**

  这个时间就是一个操作真正被你的方法进行处理的时间

####状态

比如需要按照最近一分钟、最近一小时进行数据统计的时候，之前累计的数据就可以作为状态记录。

Flink还可以创建集群的快照，一旦集群发生异常，可以从快照快速回复到最近的执行状态。

#### exactly-once、check-points

一般大数据类的系统处理一致性分成三个类别，

- at-least-once
- at-most-once
- exactly-once

之前的大部分的系统采用的都是at-least-once，因为要精确的确保exactly-once的语义往往会提高应用的复杂度和严重的拖累性能。但是flink能确保exactly-once的语义，并且几乎不会对性能造成太大的影响。

flink实现的原理就是通过check-point，将当前系统的快照储存下来，之后如果系统出错就从快照恢复。

快照算法是基于Chandy-Lamport分布式快照算法。

> 这个Lamport就是Paxos算法的作者，zookeeper的ZAB协议也是基于Paxos算法

### 一个Flink程序典型的流程

```java
// 定义执行环境
StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();

// 创建数据源
DataStream<Bill> bills = env.addSource(new MonthlyBillDataSource());

// 针对数据源进行操作
DataStream<Tuple2<Long, Long>> tuples = bills.map(new MapFunction<Bill, Tuple2<Long, Long>>() {
    @Override
    public Tuple2<Long, Long> map(Bill bill) throws Exception {
        return new Tuple2<Long, Long>(bill.getTimestamp(), bill.getTotalMoney()) ;
    }
});

// 针对处理后的结果进行落地
tuples.addSink(new WriteToCsvSink());

// 程序开始执行
env.execute("My Monthly Bill");
```

### Flink程序的部署

#### 本地模式

本地模式通常就是用来本地调试和demo学习的

####StandAlone模式

这个就是不用任何依赖，几台安装了Flink的服务器相互之间组成一个集群。

####Flink on Yarn *

Yarn是Hadoop上的一个资源管理工具，Flink on Yarn就是将Flink部署在Hadoop上，通过Yarn来管理资源

####Docker、K8S

将Flink打包成一个镜像，让容器云来调度

### Flink代码示例

maven命令快速生成一个项目template

```shell
mvn archetype:generate                               \
      -DarchetypeGroupId=org.apache.flink              \
      -DarchetypeArtifactId=flink-quickstart-java      \
      -DarchetypeVersion=1.7.2
```

#### Stream

[^见代码]: flink-share

####Batch

[^见代码]: flink-share

[Flink官方文档]: https://flink.apache.org/	"最新版本1.7"



