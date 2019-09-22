---
title: hadoop(笔记)
date: 2018-10-16 17:16:28
tags: 记录 hadoop
---



信息均来自网络，只是一个笔记，记录而已。

## 一.hadoop是什么

1.Hadoop是一个开源的框架，可编写和运行分布式应用处理大规模数据，是专为离线和大规模数据分析而设计的，并不适合那种对几个记录随机读写的在线事务处理模式。

2.Hadoop=HDFS（文件系统，数据存储技术相关）+Mapreduce（数据处理）

`关键的hadoop 2.0架构图`
![总体架构图](https://myndtt.github.io/images/4.jpg)

三大核心
分布式文件系统：HDFS —— 实现将文件分布式存储在很多的服务器上

分布式运算编程框架：MAPREDUCE —— 实现在很多机器上分布式并行运算

分布式资源调度平台：YARN——帮用户调度大量的mapreduce程序，并合理分配运算资源

<!-- more -->

## 二.HDFS

Hadoop分布式文件系统(HDFS)被设计成适合运行在通用硬件(commodity hardware)上的分布式文件系统。它能提供高吞吐量的数据访问，非常适合大规模数据集上的应用。

HDFS采用master/slave架构。一个HDFS集群是由一个Namenode和一定数目的Datanodes组成。Namenode是一个中心服务器，负责管理文件系统的名字空间(namespace)以及客户端对文件的访问。集群中的Datanode一般是一个节点一个，负责管理它所在节点上的存储。

NameNode：是Master节点，是大领导。管理数据块映射；处理客户端的读写请求；配置副本策略；管理HDFS的名称空间；

SecondaryNameNode：是一个小弟，分担大哥namenode的工作量；是NameNode的冷备份；合并fsimage和fsedits然后再发给namenode。

DataNode：Slave节点，奴隶，干活的。负责存储client发来的数据块block；执行数据块的读写操作。

![hdfs架构](http://hadoop.apache.org/docs/r1.0.4/cn/images/hdfsarchitecture.gif)



## 三.Mapreduce
MapReduce是并行处理框架，实现任务分解和调度。原理说通俗一点就是分而治之的思想，将一个大任务分解成多个小任务(map)，小任务执行完了之后，合并计算结果(reduce)。

mapreduce作业执行涉及4个独立的实体：

1.客户端（client）：编写mapreduce程序，配置作业，提交作业，这就是程序员完成的工作；

2.JobTracker：初始化作业，分配作业，与TaskTracker通信，协调整个作业的执行；

3.TaskTracker：保持与JobTracker的通信，在分配的数据片段上执行Map或Reduce任务，TaskTracker和JobTracker的不同有个很重要的方面，就是在执行任务时候TaskTracker可以有n多个，JobTracker则只会有一个（JobTracker只能有一个就和hdfs里namenode一样存在单点故障，我会在后面的mapreduce的相关问题里讲到这个问题的）

4.Hdfs：保存作业的数据、配置信息等等，最后的结果也是保存在hdfs上面
![Mapreduce架构](https://myndtt.github.io/images/6.jpg)

提到Mapreduce不得不提到hive.

Hive是在Hadoop上架了一层SQL接口（HQL作为查询接口），可以将SQL翻译成MapReduce（执行层）去Hadoop上执行，这样就使得数据开发和分析人员很方便的使用SQL来完成海量数据的统计和分析，而不必使用编程语言开发MapReduce那么麻烦。Hive的所有数据都存储在HDFS中。

![hive示意图](https://myndtt.github.io/images/61.png)

提到了hive就要提到Impala

Impala是基于Hive的大数据实时分析查询引擎，直接使用Hive的元数据库Metadata,意味着impala元数据都存储在Hive的metastore中。Hive适合于长时间的批处理查询分析，而Impala适合于实时交互式SQL查询，Impala给数据分析人员提供了快速实验、验证想法的大数据分析工具。可以先使用hive进行数据转换处理，之后使用Impala在Hive处理后的结果数据集上进行快速的数据分析。
![impala](https://myndtt.github.io/images/60.png)

注：
> Hive: 依赖于MapReduce执行框架，执行计划分成 map->shuffle->reduce->map->shuffle->reduce…的模型。如果一个Query会 被编译成多轮MapReduce，则会有更多的写中间结果。由于MapReduce执行框架本身的特点，过多的中间过程会增加整个Query的执行时间。
>
> Impala: 把执行计划表现为一棵完整的执行计划树，可以更自然地分发执行计划到各个Impalad执行查询，而不用像Hive那样把它组合成管道型的 map->reduce模式，以此保证Impala有更好的并发性和避免不必要的中间sort与shuffle

## 四.yarn

第一代Hadoop，由分布式存储系统HDFS和分布式计算框架MapReduce组成，其中，HDFS由一个NameNode和多个DataNode组成，MapReduce由一个JobTracker和多个TaskTracker组成.


第二代Hadoop，为克服Hadoop1.0中HDFS和MapReduce存在的各种问题而提出的。针对Hadoop 1.0中的单NameNode制约HDFS的扩展性问题，提出了HDFSFederation，它让多个NameNode分管不同的目录进而实现访问隔离和横向扩展；针对Hadoop 1.0中的MapReduce在扩展性和多框架支持方面的不足，提出了全新的资源管理框架YARN。

![yarn示意图](http://www.aboutyun.com/data/attachment/forum/201701/28/173203drwcdwdu31dnwdit.jpg)

可以看出JobTracker的功能被分散到各个进程中包括ResourceManager和NodeManager：比如监控功能，分给了NodeManager，和Application Master。
ResourceManager里面又分为了两个组件：调度器及应用程序管理器。也就是说Yarn重构后，JobTracker的功能，被分散到了各个进程中。同时由于这些进程可以被单独部署所以这样就大大减轻了单点故障，及压力。

![比较](https://myndtt.github.io\images\\5.jpg)

## 五.其他

### 1.Kafka
kafka是一个分布式的消息缓存系统，可以同时支持离线数据处理和实时数据处理。
### 2.spark
spark是一个非常流行的内存计算（或者迭代式计算，DAG计算）框架，在MapReduce因效率低下而被广为诟病时出现的一个令人称赞的框架。

### 3.Zookeeper

Zookeeper 分布式服务框架是Hadoop的一个子项目。它主要是用来解决分布式应用中常常遇到的一些数据管理问题，如：统一命名服务、状态同步服务、集群管理、分布式应用配置项的管理等。