---
layout: post
title: 'GFS学习笔记'
category: Notes
---

# Google File System

[The Google File System](https://github.com/boylinshan/boylinshan.github.io/blob/master/pages/gfs.pdf)

## INTRODUCTION

GFS(Goole File System) 是谷歌设计并实现的一个大规模分布式文件系统，用于支持分布式应用的数据存储需求。
在设计上GFS与其他分布式文件系统有着许多相同的特性，例如，良好的性能，高可靠性以及高可扩展性等等。此外在具体的设计上，GFS有着以下几点与早期文件系统不同的设定。
1. GFS通常运行在由成百上千台普通的机器上，硬件设备故障时常发生。因此监控，容错以及自动恢复在系统的设计中是必不可少的。
2. 在GFS使用的场景中，文件的往往比传统的文件更大，通常都是GB级别的。因此即使GFS支持KB级别文件的操作，在block大小以及IO操作的设计上，还是以GB级别作为考虑的目标。
3. 对文件来说，相对于随机写操作，追加(appending)是更为常规的操作。大多数文件在首次写入完成后，都将变成只读文件，因此追加操作将成为性能优化和原子性保证的主要目标。
4. 相对于提供通用的API，GFS在设计API时，考虑了上层应用的一些特性。因此提供了API的灵活性。

## DESGIEN OVERVIEW

## Architecture
![RPyC](/img/GFS-architecture.jpg)
GFS由一个单点master以及多个 chunkservers组成。chunkserver通常运行在普通的commodity Linux机器上。文件以大小固定的chunk为单位进行存储，chunk存储在chunkserver的本地磁盘上。每一个chunk在创建时，由master赋予一个不可变且全局唯一的64-bits chunk handle。Client通过handle以及偏移量来访问chunk中的数据。为了提高可靠性，每一个chunk都同时备份在不同的chunkserver上，数目默认为3，可配置。
Master负责管理文件系统中的所有的metadata，例如命名空间，访问控制，文件映射以及chunks的位置。同时master负责chunk lease管理，垃圾收集以及chunk的迁移等工作。简单来说，master主要负责管理chunkserver以及chunk。maseter与chunkserver通过HeartBeat消息交流。
Client仅在需要获取chunk相关信息时，才与master通信。后续的数据存储操作 ，都将直接发送给chunk所在的chunkserver。如上图所示，在首次存取数据时，Client询问Master数据chunk的handle以及location，在收到Master的回复后，Client将metadata信息缓存下来，并向相应chunkserver发送数据请求，chunkserver将所请求的数据返回给clinet。在此后的数据存取操作，若缓存的metadata未过期，则clieng不必在与master通信。在系统设计中，Master是系统中唯一的单点。因此在设计中，Master尽可能少的参与的数据的交互当中。

### Chunk Size

GFS中将一个Chunk大小设定为64MB，这比通常的文件block大了很多。使用大的chunk块，减少了client读取数据时，需要获取的chunk信息，也因此减少了与master的交互信息。此外较大的chunk块意味着较少的metedata，使得master可以将metadata存储到内存中，提高了运行的效率。然而在另一方面，大的chunk块也 意味着更容易出现负载不均衡的情况，当多个热点数据存储在一个chunk块上时，此chunk所在的chunkserver将承受大部分的访问。这种情况在GFS应用的场景中并不常见。Google通过提高replica的数量来缓解此问题。此外Google提出了允许client从client上读取数据来解决此问题的方案。

## Metadata

Master主要存储三种metadata: file和chunk的命名空间，file到chunk的映射以及chunks的位置。所有的metadata都存储在master的内存当中。同时命名空间以及映射关系也持久化到硬盘上。
将metadata存储在内存中，提高了master工作的效率，提高master可以更迅速的完成垃圾收集，chunk备份以及chunk迁移等工作，但同时也使得cluster的规模受限于master的内存大小。不过相比获得的性能提升，在出现瓶颈时，增加内存所花费的代价是可接受的。
为了提高系统的可靠性，Master将修改metadata的操作存储到log中，对于每一次修改metadata的操作，仅当将log flush到本地和远程的disk之后，才标记此次修改成功。此外为了减少failure recover的时间，Master周期性的进行记录checkpoint，checkpoint是以B-Tree的形式，将内存中的数据存储到磁盘中。再每次创建checkpoint时，为了不影响后续的操作请求，Master总是创建一个新的operation log并启用一个新的线程来完成checkpoint的创建操作。

### Consistency Model
![RPyC](/img/consistenct model.jpg)
在GFS中metadata的管理较为简单，因为metadata由单点Master进行管理。例如对于文件命名空间的修改，master通过命名空间锁来保证修改操作的原子性和正确性，并且被master按照唯一的执行顺序记录到operation log中。
而对于文件内容的修改，则较为复杂。在GFS中，将文件状态分为如下图的几种。
- consistent: 表示所有client看到的数据是一致的，无论它从那个replica上读取。
- defined: 表示所有client看到的数据是一致的，并且client可以看到所有的修改操作。
显然defined是强于consistent的一种状态，defined的文件一定是consistent的。而consistent却不一定是defined的。
所谓defined的状态，是client可以知道文件进行了什么样的修改。然而在并行的写操作的，多个client的数据被交叉的写入到了文件中，因此虽然所有client看到了相同的数据，但只能每个client自己知道自己的mutation操作写入的是什么数据，这使的region成为了consistent but undefined的状态。
成功的串行写入总使得region成为defined的状态，因为GFS保证在所有的replicas上执行相同的操作顺序，并使用chunk version来检测并排除状态错误的chunks.
当然失败的写入操作会使得region成为inconsistent的状态，因为你不知道它是在到达那个replica时失败的。

## SYSTEM INTERACTIONS

### Leases and Mutation Order
![RPyC](/img/lease_order.jpg)
为了减轻Master的工作负担，在Client进行数据操作的时候，并不直接由Master负责。对于一个Chunk的replica set，Master选出一个作为Primary Chunk来负责具体的流程。Master用lease机制来管理Primary的赋予与回收。持有lease的chunk即为Primary Chnuk。lease默认具有60秒的时效性，60秒后，chunk将不再是一个有效的primary chunk。当然chunk可以申请延长lease的时间。

在上图中，显示的client写入数据时的流程。
1. Client询问master，当前持有的lease的chunk位于哪个chunkserver以及其他replicas的位置。若当前没有chunk持有lease，则master处理完lease的赋予流程后，再将结果返回给Client.
2. Master将Primary以及Secondary Replica的位置信息返回给Client，Client缓存信息，以便于后续的使用。
3. Client将数据发送到所有的replicas上，Client可以以任意的顺序将data发送到replicas上，通过分离数据流和控制流，Client不必首先将数据发送到Primary上，而是可以以最大利用带宽的方式发送。通常ChunkServer采用全双工的方式传输数据。在理想情况下，将B字节数据传输到R个replicaes的时间是B/T + RL，T是带宽，L是两台机器间转发的延迟。
4. 当所有的relicas都回复收到数据后，Client发送写请求到Primary上，写请求附带之前所发送的数据。Primary收到写请求后，再通知其余的Secondary进行同样的操作。值得注意的时，Primary Replica可能同时收到多个Client发送的写请求，Primary Replica将所有的请求按照一定的顺序排列，并将通知其余所有的Secondary Replica按照此顺序执行写操作。从而保证了replica set的一致性。
5. Primary通知其他secondary replicas执行相同的操作。
6. Secondaries回复Primaty已经进行了相应的操作。
7. Primary将执行结果发送的client，若在任意的replicas上执行操作失败，则认为这一个请求是失败的。失败的信息会被发送到Client上。失败的操作会导致replicas set处于inconsistent的状态。GFS不负责处理失败的请求，而是通过由client再一次发送请求解决。

当Client一次性写入超过一个chunk大小的数据时，GFS会将数据拆分成多个write请求。在这种情况下，Client写入的数据可能处于consistent but undefined的状态。这是因为ChunkServerf同时面临多个Client的请求，被拆分的数据在写入时，可能会插入其他Client请求的数据。

### Atomic Record Appends

在常规的写操作中，Client指定了数据写入的位置，这在并行写入的情况下，会导致数据的写入并不是线性化的。为了保证数据写入的正常性，Client需要进行额外的获取锁以及同步等操作。为了解决这种情况，GFS提供了record append的操作，在record append操作中，Client并不指定数据的位置，而是由GFS选择，并将所选择的位置返回给Client。Record Append的操作与上图所描述的大致相同，仅有一点逻辑上更改。当Primary Replica收到写入指令时，若它发现要写入的chunk块的剩余空间小于客户端发送的数据大小，则Primary将chunk块并无效数据填满，并通知其余replicas进行相同的操作，并最后告诉Client写入失败，让Client在下一个Chunk块上继续尝试写入。为了避免过多的填充，GFS规定record append的最大数据大小为Chunk大小的四分之一。若数据可以写入，则剩余的流程与上述一致。
此外，由于replicas在收到record append操作时，会进行填充操作，因此各个replicas上的数据可能存在不一致的情况，也就是defined but unconsistent状态，这就需要Client进行数据验证的工作。

## MASTER OPERATION

### Namespace and Locking

GFS使用树形结构来管理内存中的命名空间，每一个文件和目录都是树上的一个节点。在进行读写操作时，Client需要先获取一系列的锁，才能进行后续的操作。例如，若Client需要对/d1/d2/leaf进行操作时，需要获取/d1和/d1/d2的读锁以及/d1/d2/leaf的读或写锁。读锁用来防止当前读取的节点被删除，重命名以及snapshot操作。而
写锁用来序列化创建操作。

### Creation, Re-replication, Rebalancing

GFS在三种情况下会创建一个Chunk: Creation, Re-replication, Rebalancing。当GFS创建一个Chunk时，会考虑一下几点因素。
1. 磁盘利用率：GFS优先选取磁盘利用率低的chunkserver创建新的chunk.
2. 历史数据：GFS限制某一个chunkserver最近创建chunk的数目来避免chunkserver后续承受过高的复杂.
3. 可靠性：为了确保数据的可靠性，GFS希望将chunks分布到不同的racks上.

Master在chunk的可用replicas数据低于下限时，进行Re-replication操作。在需要进行多个Re-replication操作时，Master的原则是首要满足阻塞Client的那些replicas set的需求。而rebalances则是master周期性进行的一项操作，用来进行负载均衡。

### Garbage Collection

在GFS中，当应用删除文件时，Master立刻记录删除文件操作的log，但是并不立刻进行文件的删除，而是将文件名改成另一种格式（包括删除的时间），并在后续的常规扫描中，删除删除时间拆过默认配置的3天的已删除文件。在此之前，文件仍然是可操作的，并且可以通过重名命将文件重新添加的文件系统中来。同样的在扫描时，Master将删除系统中的孤儿chunk，也就是在file-to-chunk的映射表中，没有任何文件映射到的chunk。Master将此信息通过HeartBeat信息发送给Chunkserver。也就是在HeartBeat中，Chunkserver发送自己所拥有的chunks，而Master的回复中标记出，已经不在自己metadata，也就是被删除的chunks。Chunkserver收到此信息后，便回收掉这些chunks所使用的disk资源。

### Stale Replica Detection

在更新数据的过程中，ChunkServer时常处于failed的状态，因此ChunkServer上的数据可能处于stale的状态。为了解决这一问题，GFS在操作数据的过程中，都附带一个chunk version number，用于验证当前chunk的状态。Master在每次赋予lease的时候递增chunk version number的值，并通知所有chunk replicas更新相应的数值。当ChunkServer由于failed而没有收到更新时，对应的chunk的version number将低于Master上所存储的数据。因而在循环的 HeartBeat信息中，Master就能够判断哪些chunk已经处于stale的状态，从而在下一次的Garbage Collection中将其回收掉。

## FAULT TOLERANCE AND DIAGNOSIS

可以看到，GFS始终认为底层机器处于failure的状态，因此在GFS的设计中，使用了许多策略来解决这一问题。
- Fast Recovery: Master和ChunkServer都被设计成可以快速恢复状态的形式，因此它们并不区分正常和非正常的重启。
- Chunk Replication:  Chunk被备份在多个ChunkServers以及多个Racks上。由Master负责确保正常可用的备份数始终处于配置的数量。
- Master Replication:  Master也采用的备份的手段，但metadata数据的修改始终由primary master负责，其他备份仅在primary master失败时，提供read only的功能。当primary master机器出问题时，由底层管理系统负责重新调度。为此，Client采用别名访问master，别名被设计为DNS的形式。
- Data Integrity:  为了探测disk损坏，每一个ChunkServer都为自己上的chunk维护一份checksum，当client读取数据时，ChunkServer首先计算checksum时候正确，在正确的情况下才返回数据。若checksum错误，则报告master，由master处理后续流程。
