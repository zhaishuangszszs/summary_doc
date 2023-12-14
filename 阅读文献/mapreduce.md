# MapReduce:在大集群上简化数据处理

## 摘要
&emsp;&emsp;MapReduce是一个编程模型,和一个相关实现对于处理,产生大数据集。用户指定一个map函数处理一个key/value对,从而产生中间的key/value对集.然后再指定一个reduce函数合并所有的具有相同中间key的中间value。如本文所示，许多现实世界的任务都可以用该模型表示。

&emsp;&emsp;用这种函数式风格编写的程序可以自动并行化，并在大型商用机器集群上执行。运行时系统负责对输入数据进行分区、在一组机器上调度程序的执行、处理机器故障以及管理所需的机器间通信等细节。这使得没有任何并行和分布式系统经验的程序员可以轻松地利用大型分布式系统的资源。

&emsp;&emsp;我们的MapReduce实现运行在一个大型的商用机器集群上，并且具有高度的可扩展性:一个典型的MapReduce计算在数千台机器上处理许多TB（terabytes）的数据。程序员发现这个系统很容易使用:数百个MapReduce程序已经实现，每天在谷歌的集群上执行超过1000个MapReduce任务。

## 1 Introduction
&emsp;&emsp;在过去的5年里,作者和Google的许多人已经实现了数以百计的为专门目的而写的计算来处理大量的原始数据,比如,爬行（crawler）的文档,Web请求日志,等等.为了计算各种类型的派生数据,比如,倒排索引,Web文档的图结构的各种表示,每个主机上爬行的页面数量的概要,每天被请求数量最多的集合,等等.很多这样的计算在概念上很容易理解.然而,输入的数据量很大,并且只有计算被分布在成百上千的机器上才能在可以接受的时间内完成.怎样并行计算,分发数据,处理错误,所有这些问题综合在一起,使得原本很简介的计算,因为要大量的复杂代码来处理这些问题,而变得让人难以处理。


## 2 Programming Model


## 3 Implementation

### 3.3 Fault Tolerance

#### Master Failure
让master写入上述master数据结构的定期检查点是很容易的。如果master任务终止，则可以从最后一个检查点状态开始新的副本。然而，考虑到只有一个master，它不太可能失败;因此，如果master服务器失败，我们当前的实现会中止MapReduce计算。客户端可以检查这种情况，如果需要的话，可以重试MapReduce操作。

#### Semantics in the Presence of Failures
当用户提供的map和reduce操作是其输入值的确定性函数时，我们的分布式实现产生的输出与整个程序的非错误顺序执行所产生的输出相同。
我们依靠map的原子提交和reduce任务输出来实现这个属性。**每个正在进行的任务将其输出写入私有临时文件**。一个**reduce任务**生成**一个**这样的文件，一个**map任务**生成**R个**这样的文件(**每个reduce任务生成一个**)。` A reduce task produces one such file, and a map task produces R such files (one per reduce task).`**当map任务完成时**，worker向master**发送一条消息**，并在消息中**包含这R个临时文件的名称**。`When a map task completes, the worker sends a message to the master and includes the names of the R temporary files in the message.`如果主机接收到**已经完成的map任务**的**完成消息**，则**忽略该消息**。否则，它将在**主数据结构中记录这R个文件的名称**。
当reduce任务完成时，reduce worker**自动将其临时输出文件重命名为最终输出文件**。如果在多台机器上执行**相同的reduce任务**，则将对相同的最终输出文件**执行多个重命名调用**。我们依赖底层文件系统提供的原子重命名操作来保证最终的文件系统状态**只包含一次reduce任务执行所产生的数据**。
绝大多数map和reduce operator都是确定性的，在这种情况下，我们的语义和**顺序执行**事实上等价，这使得程序员很容易对他们的程序行为进行推理。当map and/or reduce operator 不确定时，我们提供较弱但仍然reasonable语义(semantics)。在存在非确定性operator的情况下，**特定reduce任务R1的输出**相当于**非确定性程序**的**顺序执行对R1产生的输出**。然而，**不同的(non-deterministic)reduce任务R2**的输出可能**对应于不同顺序执**行所产生的R2的输出。`However, the output for a different reduce task R2 may correspond to the output for R2 produced by a different sequential execution of the non-deterministic program.`
考虑map任务M和reduce任务R1和R2。设e(Ri)为所提交的Ri的执行(只有一次这样的执行)较弱的语义(**weaker semantics**)出现是因为e(R1)可能读取了一次M执行产生的输出，而e(R2)可能读取了另一次M执行产生的输出。

### 3.4 Locality
网络带宽在我们的计算环境中是一种相对稀缺的资源。我们节省网络带宽通过利用这一事实，即数据(由GFS[8]管理)存储在组成集群的机器的本地磁盘上。GFS将每个文件划分为64MB的块，并在不同的机器上存储每个块的几个副本(通常是3个副本)MapReduce master**考虑输入文件的位置信息，并尝试在包含相应输入数据副本的机器上调度地图任务**。**如果失败，它会尝试在该任务输入数据的副本附近调度一个map任务**(例如，在与包含数据的机器位于同一网络交换机`same network switch`上的工作机器上)。**当运行较大MapReduce operations用到集群相当多的worker时，大部分输入数据在本地读取，不消耗网络带宽。**

### 3.5 Task Granularity（任务粒度）