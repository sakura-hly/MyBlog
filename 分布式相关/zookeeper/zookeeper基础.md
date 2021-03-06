# Zookeeper基础
Zookeeper是一个开源的分布式协调服务，其设计目标是将那些复杂且容易出错的分布式一致性服务封装起来，构成一个高效可靠的原语集，并以一系列简单易用的接口提供给用户使用。

## Zookeeper是什么
Zookeeper是一个典型的分布式数据一致性的解决方案，分布式应用程序可以基于它实现诸如数据发布/订阅、负载均衡、命名服务、分布式协调/通知、集群管理、
Master选举、分布式锁和分布式队列等功能。Zookeeper可以保证如下分布式一致性等特性。

1. 顺序一致性

   从同一个客户端发出的事务请求，最终将会严格地按照其发起顺序被应用到Zookeeper中去。
2. 原子性

   所有事务请求的处理结果在集群所有机器上是一致的，要么整个机器所有机器都成功应用了该事务，要么都没有应用。一定不会出现部分机器应用了该事务，另外一部分没有应用的情况。
3. 单一试图

   无论客户端连接的是哪一台Zookeeper服务器，其看到的服务器数据模型都是一样的。
4. 可靠性

   一旦服务端成功应用了一个事务，并完成对客户端的响应，那么该事务引起的服务器状态变更将会被一致保留下来，除非有另一个事务又对其做了更改。
5. 实时性

   Zookeeper仅仅保证在一定的时间段内，客户端最终一定能够从服务器上读取到最新的数据状态。

## Zookeeper的设计目标
1. 简单的数据模型

   Zookeeper使得分布式程序能够通过一个共享的、树形结构的名字空间来进行相互协调。这里所说的树形结构的名字空间，是指Zookeeper服务器内存中的一个数据模型，
   其由一系列被称为ZNode的数据节点组成，总的来说，其数据模型类似于一个文件系统，而ZNode之间的层级关系，就像文件系统的目录结构一样。不过和传统的磁盘文件系统不同，
   Zookeeper将全量数据存储在内存中，以此来实现提高服务器吞吐、减少延迟的目的。
2. 可以构建集群

   一般3~5台机器就可以组成一个可用的Zookeeper集群了。集群中的每台机器都会在内存中维护当前的服务器状态，并且每台机器之间都互相保持通信。值得一提的是，
   只要集群中存在超过一半的机器能够工作，那么整个集群就能够正常对外服务。
   
   Zookeeper的客户端程序会选择和集群中任意一台机器共同来创建一个TCP连接，而一旦客户端和某台Zookeeper服务器之间的连接断开后，客户端会自动连接到集群中的其它机器。
3. 顺序访问

   对于来自客户端的每个更新请求，Zookeeper都会分配一个全局唯一的递增编号，这个编号反映了所有事务操作的先后顺序，应用程序可以使用Zookeeper的这个特性来实现更高层次的同步原语。
4. 高性能

   由于Zookeeper将全量数据存储在内存中，并直接服务于客户端的所有非事务请求，因此它非常适用于以读操作为主的应用场景。
   
## 集群角色
Zookeeper没有沿用传统的Master/Slave概念，而是引入了Leader、Follower和Observer三种角色。Zookeeper集群的所有机器通过一个Leader选举过程来选定一台被称为“Leader”的机器，
Leader为客户端提供读写服务。除Leader外，其它机器包括Follower和Observer。Follower和Observer都能够提供读服务，区别在于Observer不参与Leader选举过程，也不参与写操作的“过半写成功”策略。
因此Observer可以在不影响写性能的情况下提升集群的读性能。

## 会话（Session）
Session是指客户端会话。在Zookeeper中，一个客户端连接是指客户端与服务器之间的一个TCP长连接。Zookeeper的对外服务端口默认是2181，客户端启动的时候，
首先会与服务器建立一个TCP连接，从第一次连接建立开始，客户端会话的生命周期也开始了，通过这个连接，客户端能够通过心跳检测与服务器保持有效的会话，
也能够向Zookeeper服务器发送请求并接收响应，同时还能够通过该连接接收来自服务器的Watch事件通知。

Session的sessionTimeout值用来设置一个客户端会话的超时时间，当由于服务器压力过大、网络故障或客户端主动断开连接等各种原因导致客户端连接断开时，
只要在sessionTimeout时间内能够重新连接上集群任意一台服务器，那么之前创建的会话仍然有效。
## 数据节点（ZNode）
Zookeeper中节点分为两类，第一类是构成集群的机器，称为机器节点；第二类是数据模型中的数据单元，称为数据节点——ZNode。Zookeeper将所有数据存在内存中，
数据模型是一棵树（ZNode Tree），由斜杠（/）进行分割的路径，就是一个ZNode，例如/foo/path1。每个数据节点上都会保存自己的数据内容，还有一系列属性信息。

在Zookeeper中，ZNode还可以分为持久节点和临时节点。所谓持久节点就是指一旦这个ZNode被创建了，除非主动进行ZNode的移除操作，否则这个ZNode将一直保存在Zookeeper中。
而临时节点的生命周期和客户端会话绑定，一旦客户端会话失效，那么这个客户端创建的临时节点都会被移除。另外，Zookeeper还允许用户为每个节点添加一个特殊的属性：SEQUENTIAL。
一旦节点标记上这个属性，那么在这个节点被创建的时候，Zookeeper会自动在其节点名后面追加上一个整型数字，整个整型数字是一个由父节点维护的自增数字。
## 版本
对于每一个ZNode，Zookeeper都会为其维护一个叫做Stat的数据结构，Stat中记录了这个ZNode的三个数据版本，分别是version（当前ZNode的版本）、cversion（当前ZNode子节点的版本）
和aversion（当前ZNode的ACL版本）。
## Watcher
Watcher（事件监听器），是Zookeeper一个很重要的特性。Zookeeper允许用户在指定节点上注册一些Watcher，并且在一些特定事件触发的时候，Zookeeper服务端会将事件
通知到感兴趣的客户端上去，该机制是Zookeeper实现分布式协调的重要特性。
## ACL
Zookeeper采用ACL（Access Control Lists）策略来进行权限控制，类似于UNIX文件系统的权限控制，Zookeeper定义了以下五种权限：
1. CREATE：创建子节点的权限
2. READ：获取节点数据和子节点列表的quanx
3. WRITE：更新节点数据的权限
4. DELETE：删除子节点的权限
5. ADMIN：设置节点ACL的权限

需要注意的是：CREATE和DELETE都是针对子节点的权限控制。