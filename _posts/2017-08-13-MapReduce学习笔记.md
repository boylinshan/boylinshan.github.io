---
layout: post
title: MapReduce学习笔记
category: Notes
---

# MapReduce学习笔记

[MapReduce: Simplified Data Processing on Large Clusters](https://github.com/boylinshan/boylinshan.github.io/blob/master/pages/mapreduce.pdf)

##  Introduction

MapReduce是Google提出的一个分布式计算模型，应用于大规模数据的并行处理场景。在现实应用中存在着一些处理逻辑较为简单，但数据量较为庞大的问题。这一类的问题的难度不在于数据处理逻辑，而在于如何利用成百上千台机器同时处理数据，如何划分数据以及如何应对机器故障等问题。在Google提出的MapReduce框架中统一处理了上述问题，并为使用者提供了_map_和_reduce_两个函数接口设计具体的计算逻辑。

## Programming Model

MapReduce的计算模型主要由map和reduce两部分组成。
- map函数处理输入的数据，将数据处理成key/value键值对的形式输出成中间数据以供reduce函数使用。
- reduce函数接收多个map函数处理后的中间数据，MapReduce库将同一key值的多份中间数据发送给一个reduce进程上由reduce函数处理并输出。一般而言reduce函数数据一份或零份数据，此数据可能作为最终的数据，也可能成为下一次map-reduce流程的输出数据。

计算多个文件中每个单词出现的频率的map和reduce函数示例代码如下。
```python
	def map(key, value):
	'''
	:param key: document name.
	:param value: document contents.
	'''
	for word in contents:
		yield (w, '1')
	
	def reduce(key, values):
	'''
	:param key: a word.
	:param values: a list of counts.
	'''
	result = 0
	for value in values:
		result += eval(value)
	return result
```
此外MapReduce计算模型还可以应用到以下几个场景.
- Distributed Grep:  Map函数输出输入文件中符合匹配规则的行，Reduce函数将所有Map函数输出汇总的最终文件中.
- Count of URL Access Frequency:  Map函数处理输入的web log，将访问log以<URL, 1>的格式输出，Reduce函数输入同一URL的所有访问记录，然后汇总成<URL, totol count>的格式输出到最终结果文件.
- Reverse Web-Link Graph:  对于在一个名为source的page中每一个的链接， Map函数都以<target, source>的形式输出次链接，其中target是链接的目标地址。Reduce函数接收同一target的所有source urls，并将所有输入以<target, list(source)>的形式输出.
- Term-Vector per Host: term vector用于总结文档中的关键字，Map函数统计文档中的字出现的频率，以<word, frequency>的形式存放在term vector中。最后以<hostname, term vector>输出，hostname是根据文档URL提取出的字段。Reduce函数接受同一hostname的所有输出，并将它们term vector中的<word, frequency>合并，删选掉出现频率不够高的字段，然后输出最终结果.
- Inverted Index:  对于文档中的字段，Map函数以<word, document ID>的形式输出，最终Reduce函数叫中间结果合并成<word, list(docutment ID)>的形式。
- Distributed Sort:

## Implementation

### Environment
MapReduce在不同环境中的实现方式不尽相同。当机器性能较强时，可以将多个map和reduce进程放置在同一台机器上。而当机器性能一般时，则可以将map和reduce进程放置在不同的机器上，利用集群的能力快速的处理数据。
Google论文中所描述的环境为使用以太网连接的大量商用PC组成的计算集群，也就是第二种情况。因为Google使用分布式的MapReduce实现方式来处理问题，Google的计算环境如下若述。
1. 集群中的机器多为双核的X86处理器、Linux系统、2-4GB的内存。
2. 机器间连接的网络带宽为 100MB/S 或 1GB/S
3. 集群由成百上千台机器组成，因此机器出错是常见的情形。
4. 机器使用IDE存储数据，底层数据存储在分布式文件系统当中。
5. 用户以jobs的形式提交到调度系统(Borg)，每一个job包含多个tasks，而每一个task则代表一个map或reduce任务。

### Execution Overview
在Google的执行流程中，Map调用将用户提供的map函数分布到到M台机器上并自动的将数据分成M份，随后开始并行的处理数据，每个map函数产生R份输出文件，R为reduce函数的数量。Reduce调用将用户提供的reduce函数分布到R台机器上，并等待处理map函数处理后的数据，每个reduce函数产生1个或0个输出文件，数字R和M由用户指定。下图展示了MapReduce操作的整个流程。
![RPyC](/img/mapreduce.jpg)

1. MapReduce库首先将数据分割成M份大小在16MB-64MB大小的数据块(GFS chunk)，然后在多台机器上开始执行。
2. MapReduce库在开始执行流程时，会启动一个master进程，此master进程负责将M个map任务以及R个reduce任务部署到空闲worker上。
3. 部署map任务的worker读取相应的输入数据块，并使用用户定义的map函数处理数据，最后将处理好的中间数据缓存到内存中。
4. map worker周期性的将中间数据存储的磁盘中，并根据key值将数据分隔成R份，每一个对应一个reduce worker。最终map worker将中间数据的位置传回到master上，由master通知reduce worker数据位置。
5. reduce worker收到master传递的数据位置信息后，使用RPC读取中间数据。当其读取到所有的中间数据后，将数据进行排序并随后使用用户提供的reduce函数开始数据处理的流程。
6. reduce worker遍历中间数据集，并将相同key值的所有数据传递给reduce函数。reduce函数处理后，将数据写入到最终结果文件。
7. 当所有的map和reduce任务完成后，master唤醒user应用。此时，整个MapReduce调用已经完成。

一个reduce任务会产生一份最终输出，因此在整个调用成功的完成后，会产生R份最终输出文件。

### Master Data Structures

Master作为MapReduce的中心节点，存储了一些数据以完成整个流程。对于每一个map和reduce任务，Master存储它的状态(idle, in-progress, completed) 以及 worker机器的ID(non-idle tasks)。
Master作为中间数据的中转站，在每一个map任务完成时，都会将输出的R份中间数据的位置以及大小发送到Master上，Master再将此信息推送到in-progress的reduce任务上。

### Fault Tolerance

#### Worker Failure

Master周期性的ping运行着任务的work，如果在一段时间内没有回应，Master将worker标记为failed。任何完成的map task被标记为idle状态(意味着可以被再次调度)，任何在failed worker上的map和reduce任务也被标记为idle状态，因此也可以再被调度。
由于map任务输出的中间数据存储在worker本地，因此failed worker上的map任务需要再被执行。而reduce任务输出的最终数据存储在GFS中，因此一旦reduce任务完成，则不需要再一次的调度。此外当map任务重新执行时，所有的reduce任务都会收到此信息，并开始从新的map worker读取数据。

#### Master Failure

Master可以使用checkpoint的方式来保存执行的状态，然而在本文的实现中，并不采用此种方式。而是在Master失败的情况下通知client，并由Client决定后续的处理方式(放弃or重新执行)

#### Semantics in the Presence of Failures

当用户的map以及reduce操作的输出是确定性的情况下， MapReduce调用保证其处理输出结果与顺序执行的结果相同。MapReduce使用原子性提交来保证此语义。
由于map函数的输出存储在本地，因此Master只要保证不接受已经完成过的map任务的输出就已经足够了。而reduce函数的输出存储在GFS中，因此每一个reduce任务都将数据暂时的存储在一个临时文件中，等数据处理完毕后，再将临时文件改名为最终输出文件。因为在GFS中，文件的改名操作是具有原子性保证的，因此这也就保证了不存在多个相同的reduce任务输入数据到一个最终文件中的情况。

#### Locality and Task Granularity

在Google的运行环境中，带宽是较为稀缺的资源。因此Master在调度map任务时，首先考虑将mak任务放置到存储数据的worker上(在GFS中，数据以64MB大小的chunk存储，并默认备份三份)，或者尽量靠近存放数据的worker已减少带宽使用。
在上述MapReduce的执行过程中，启动了M个map任务，R个reduce任务以及一个Master，而Master进行了O__(M+R)__次调度，并在内存中存储__O(M*R)__大小的状态数据。在实际应用中，M的数量往往大于可用的worker以便于更好的负载均衡以及故障恢复。而R的数量则经常由用户指定。文中给出一组数据为__M=200,000, R=5,000, work=2,000__

#### Backup Tasks

一个MapReduce的执行过程，往往会因为最后几个任务而拖慢整体的速度。针对这种情况，MapReduce在接近完成的时候，会针对未完成的task在不同的worker上启动一个backup任务，backup和normal任务的完成都标志着任务的完成，此机制可以很好的提高MapReduce的速度。

### Refinements

#### Partitioning Function

MapReduce用户在处理数据时，可以提供特定的键值对分发方式。在MapReduce的默认实现中，使用hash函数来进行分配。也就是对于R个reduce任务而言，默认的分隔函数为__hash(key) mod R__. 用户可以更改此默认的分隔函数，来满足特定的需求。

#### Ordering Guarantees

在MapReduce的实现中，默认实现了同一partition分区中，所有键值对的排序功能。也就是说，Reduce任务接收到的中间临时数据是有序的。

#### Combiner Function

在有些MapReduce的应用中，对部分数据进行merge操作可以优化整个MapReduce调用的进行，因此MapReduce库为用户提供了一个Combiner的接口。Combiner函数运行在执行Map任务的worker上，在Map函数处理完数据之后，将数据存储到本地文件之前执行。通常而言，Combiner函数采用与Reduce函数相同的代码，区别只是在于，Reduce函数的输出存储到最终的输出文件，而Combiner函数的输出存储到中间临时文件中。

#### Input and Output Types

  Input 和 Output的类型用来决定输入数据和输出数据时的行为。例如，MapReduce提供了__text__模式来读取数据，在__text__模式下，数据以行为单位读取数据，key/value分别时每一行的偏移量一集此行的内容。通过MapReduce提供的接口，用户可以定义自己的输入输出类型。

#### Counters

MapReduce库提供了一个Counter类型的数据来进行全局的统计功能。用户在编写Map or Reduce函数是，可以声明一个Counter类型的变量统计某些数据，此数据被在worker和master的心跳包中，被发送的Master上，由Master进行全局的统计，并由Master保证同一Counter变量，全局统计的正确性。