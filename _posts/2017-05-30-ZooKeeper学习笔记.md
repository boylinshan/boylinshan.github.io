---
layout: post
title: 'ZooKeeper学习笔记'
category: Notes
---

## ZooKeeper学习笔记

[ZooKeeper: Wait-free coordination for Internet-scale systems](https://github.com/boylinshan/boylinshan.github.io/blob/master/pages/Zookeeper.pdf)

### Introduction

Zookeeper是一个开源的状态同步系统，它经常被集成到其他分布式应用中处理集群中状态同步方面的问题。不同于其他状态同步系统，Zookeeper并不专注于解决某一种问题，而是提供了更加底层的接口。利用这些接口，应用可以实现诸如group messageing，configuration management，distributed lock以及leader elect等方面的问题。除此之外，ZooKeeper还具有wait-free的特性，即服务并不会被当前的请求阻塞中，这就避免了服务的性能被slow或faulty客户影响的问题，然而请求完全的并行不能完成状态同步的需求，因此ZooKeeper在对于数据的操作上，提供了以下两点保证。
- FIFO：Zookeeper保障同一个客户端发出请求的顺序和请求到达的顺序相同。
- Linearizability：Zookeeper保证所有更改数据状态的操作线性执行。

ZooKeeper在网络层使用了TCP/IP协议，因此便将保证FIFO的工作交给了TCP/IP协议来完成，减轻了Zookeeper的复杂性。
对于Linearizability而言，Zookeeper自身实现了一个atomic broadcast协议Zab，来保证此点。其实这一点是针对write操作而言的。由于Zookeeper自身也是使用replica来保证服务的可用性，所以在replica set中的某一个server收到write请求时，需要同步set中所有servers的状态，来保证服务的正确性。对于读操作，server在本地就处理了。
此外Zookeeper还允许Client缓存某些数据，避免了Client每次需要数据时都要先向Server发送请求。Server在每次数据变化时，向感兴趣的client发送消息过期的信息。简单来说，即是对于服务器信息的处理方式，比较同意想到的方式有两种。
1. 完全存放在服务器，客户端每次需要时再获取。
2. 缓存在客户端，过期时由服务器发送到客户端，客户端请求时可直接使用。

在不同的情况下，这两种方式互有优劣。
1. 客户端请求频繁，服务器变更不频繁：很显然，在这种情况下，方式2比较高效。
2. 服务器变更频繁，客户端请求不频繁：这种情况，方法1占优。

ZooKeeper使用了类似于方式2的做法， 但是采用了更加高效的做法。可以看到，在第二种情况下，方法2之所以不高效的原因时，服务器需要频繁的同步数据的状态，以保障客户端上的信息可用。然而这一点是没有必要的，服务器仅需将信息过期这个消息告诉客户端即可，后续的操作由客户端决定。其实在学习的时候也发现，为了保证自身的高效性，Zookeeper将很多事情从server上分离了出来，仅仅实现了最核心，最重要的那部分功能。

### The ZooKeeper service

#### overview

ZooKeeper以类似于UNIX文件系统结构的方式来管理状态数据。不同的是，所有的数据都存在内存中，定期的在硬盘上做备份。
![RPyC](/img/zknamespace.jpg)

UNIX文件系统中，每一个文件或者目录都是一个inode，而在ZooKeeper中，数据则是以ZNode的形式表示。Client通过Zookeeper提供的API，操控这些Znode。Client可以创建以下两种类型的Znode。
1.  Regulaer：常规的Znode，客户端需要显式的创建和删除node。
2.  Ephemeral：此种Znode与客户端的session相关联，当创建他们的session终止的，node被删除。可以想象成跟session有着一种weakref的关系。

此外在创建znode时，Client可以设置sequential标记，标记会被添加到znode名字的后面。任何一个子节点的sqquential标记都不小于其父节点。
Client还可以设置 watch标记，watch标记相当于在server上注册了一个一次性的触发器，当watch的数据发生变化时，server将消息发送到对应的客户端。

#### Client API

这里记录一下ZooKeeper中一些常用的API，并没有罗列所提供的所有API。

- create(path, data, flags): 在path下创建一个znode，并存储data[]数据，flags用于标记znode的类型以及sequential标记。
- delete(path, version): 如果当前znode的version等于给定version，则删除路径为path的znode。
- exists(path, watch): 判断path下是否存在znode。watch为True，则监控path的状态变化。
- getData(path, watch): 返回路径path下znode的数据，并在znode存在的情况下，监控path的状态变化。
- setData(path, data, version): 如果path下znode的version等于给定version，则将data[]写入到znode中。
- getChildren(path, watch): 返回路径path下znode的所有子znode。
- sync(path): 通知所连接的server等待当前sets中所有的更新操作。

上述操作提供synchronous和asynchrounous两种形式。sync会block住，async等执行结果返回后执行相应的操作。
Zookeeper并不适用handler来记录连接，每次都使用full path来访问znode。

### Examples of primitives

#### Configuration Management

使用ZooKeeper提供的API，我们可以完成分布式环境中配置管理的功能。Master Process启动时将配置文件存储在一个znode上面。Work Processes启动时，去相应的路径下读取配置数据并将watch设置为True，这样当配置文件更改时，Work Processes将会收到通知。

#### Group Membership

由于ZooKeeper使用了类UNIX文件系统的结构来存储数据，应用可以很方便完成群组管理的功能。用UNIX文件系统的概念来解释的话，一个目录就代表着一个群组，目录下的一个文件便代表一个成员。每个成员都可以获取到其他成员的信息，而目录则存储着整个群组的数据。此外通过创建Ephemeral类型的结点，群组管理还可以监控组里成员的状态。

### ZooKeeper Implementation

![RPyC](/img/zkcomponents.jpg)

上图是ZooKeeper的基本结构。当请求到达时，由Request processor进行处理。Request Processor将write请求转发到leader上，而read请求则由client所连接的server自行处理。
Replicated Database用于存储Zookeeper保存的数据，是一个in-memory数据库。write操作在写入到数据库前会在写到disk上。
