# Google File System(GFS)

[go back to home](/)

[Paper link](https://research.google/pubs/pub51/)

Proceedings of the 19th ACM Symposium on Operating Systems Principles, ACM, Bolton Landing, NY (2003), pp. 20-43

Last Edit: Jan 18, 2023

---

GFS 有非常多的东西，这里只写了一些重要的部分。像是snapshot，文件删除，高可用机制，Replica管理等没有具体提及。

# Introduction

GFS是google提出的一个可扩展分布式文件系统，为大型分布式数据密集型应用提供服务。可以在大规模的消费级机器集群上提供不错的容错能力。GFS在设计的时候主要依据6个假设(观察得出的):

1. **节点失效经常发生**。系统由非常多的消费级机器组成，大量用户同时进行访问，这使得节点很容易因为程序bug、磁盘故障、内存故障等原因失效。
2. **存储以大文件为主**。每个文件通常100MB或几GB。系统需要支持小文件，但不需要对其进行特殊的优化。
3. **大容量连续读，小容量随机读取**是文件系统中的常态。
4. **写入也已大容量为主，小容量无需特殊优化。**
5. **支持原子的文件追加操作**。使得大量用户可以并行追加文件，而不需要额外的加锁机制。
6. **高吞吐量比低延时更重要**

> **为什么设计一个优秀的分布式存储系统非常的困难:**
> 
> 1. Performance. 当数据量非常大的时候，数据分片(Sharding)是非常重要的。
> 2. Fault Tolerance. 但是由于数据在多台服务器上分片。由于多台服务器，整个系统出现故障的概率会大很多。因此需要容错机制
> 3. Replication. 复制数据副本到多台服务器上，但是为了用户能够拿到一致的数据，需要考虑一致性
> 4. Consistency. 为了一致性，不得不使用网络(极慢的数据交互方式)进行确认和同步。这又会影响性能(Performance)!!!
>
> 世界因此变成了一个美妙的环。:-)。这也是分布式的挑战之处。

GFS讨论了上述的这些主题和在实际生产场景中的应用。

# Architecture

<div align="center"> 
<img src="./imgs/GFS-arch.png" width = "60%"/>
<br>
    <div style="color:orange; border-bottom: 1px solid #d9d9d9;
    display: inline-block;
    color: #999;
    padding: 2px;">Fig 1. GFS Arch[1]</div>
</div>

一个GFS集群由一个Master节点和若干个Chunk Server节点组成。每个Chunk Server可以被许多个Client访问。GFS Chunk Server作为用户级进程在Linux服务器中运行，并且文件系统本身使用的就是Linux系统的那套。(所以文中说，没有特地的为GFS Chunk Server加入cache功能，因为Linux文件系统已经干了这件事)

Chunk: GFS中的文件在存储的时候会被分割成多个Chunk，每个Chunk的大小为64MB。在Chunk分配的时候，Master会分配一个Handle给Chunk，类似于指针。Chunk在google的实现中，使用3份副本。

Master: 维护元数据，记录文件被分割为哪些Chunk、以及这些Chunk的存储位置；它还负责Chunk的迁移、重新平衡(rebalancing)和垃圾回收；此外，Master通过心跳机制与ChunkServer通信，**向其传递指令(是的，也是通过心跳机制)**，并收集状态；

Client: 首先向Master询问该文件的Chunk在哪里，Chunk Server位置，再从 Chunk Server获取数据。Client并不会每次都向Master发送数据进行询问，它会cache一部分的数据(不是Chunk的，是某个文件对应的Chunk Server的位置，即Chunk的地址)，并且保持一定的时间。

ChunkServer: 存储Chunk，Client和Chunk Server不会缓存Chunk数据，防止数据出现不一致

## Chunk and Chunk Size

Chunk大小的选择是非常重要的。通常情况下，Chunk大小为64MB，这比典型的文件系统的block大得多，可以通过惰性空间分配策略，来避免因内部碎片造成的空间浪费。

不过，较大的Chunk会使得小文件占据额外的存储空间。此外，小文件通常只会占据一个Chunk，当大量客户端访问这个小文件时，这个Chunk容易成为系统的负载热点。

Chunk的大小设置主要考虑这些因素：

1. 减少Master保存的元数据大小(每个chunck都对应着一份元数据，分割的Chunk太多会导致元数据非常的大。类似的，可以比作内存分页中的页表)，使得可以把元数据全部放在内存中。
2. 减少Client与Master的通信次数，因为对同一个Chunk的多次读写只需要请求一次Chunk信息(Client会暂时的存储这些元信息，类比于内存的Spacial locality特性)。
3. 增大Client操作落到同一个Chunk上的概率。通过保持持久的TCP连接来减少网络上的负载。(类似的，可以类比于内存的Temporal locality特性)

## Master

为了简化设计，只有一台机器会作为Master存在。Master在内存中存储3种metadat。标记 nv(non-volatile, 非易失) 的数据需要在写入的同时存到磁盘(为了效率，也有批写入的)，标记v的数据，Master会在启动后查询Chunk Server 集群。

1. namespace(目录层次结构)和文件名(nv)

2. 文件名 -> array of Chunk Handles 的映射(nv)

3. Chunk Handles -> 版本号(nv)、list of Chunk Servers(v)、primary(v)、lease(v)

Master 使用Log和CheckPoints来进行记录而不是使用数据库的形式。Log形式可以在尾部快速添加，相比于数据库的复杂数据结构，Log的速度会更快。并且Log是一种顺序的结构，可以很好的体现出时间线。

### 元数据管理[2]

元数据保存在Master内存中使得Master要对元数据作出变更变得极为容易；同时，这也使得Master可以简单高效地周期性扫描整个集群的状态，以实现Chunk回收、迁移、均衡等操作。一个64MB的chunck需要使用掉64KB的空间来存储元数据。

Master会把前两类信息(namespace+文件名，文件名到chunck Handles的映射)以日志形式持久化存储在Master的本地磁盘上，并在在其他机器上备份，但是不会持久化保存Chunk Replica的位置信息，而是在集群启动时由Master询问各个Chunk Server其当前所有的Repica。这样做可以省去由于Chunk Server离开集群、更改名字、重启等原因的Master与Chunk Server的同步问题。此后，Master通过心跳包来监控Chunk Server的状态并更新内存中的信息。

为了保证元数据的可用性，Master在对元数据做任何操作前对会用先写日志的形式将操作进行记录，只有当日志写入完成后才会响应客户端的请求，而这些日志也会备份到多个机器上。日志不仅是元数据的唯一持久化记录，也是定义操作执行顺序的时间线。文件、Chunk和他们的版本信息都由他们的创建时间唯一的永久的标识。

### Namespace管理[2]

在逻辑上，Master并不会根据文件与目录的关系以分层的结构来管理这部分数据，而是单纯地将其表示为从完整路径名到对应文件元数据的映射表，并在路径名上应用前缀压缩以减少内存占用。

为了管理来自不同客户端的并发请求对Namespace的修改，Master会为Namespace中的每个文件和目录都分配一个读写锁（Read-Write Lock）。由此，对不同Namespace区域的并发请求便可以同时进行。

所有Master操作在执行前都会需要先获取一系列的锁：通常，当操作涉及某个路径 /d1/d2/.../dn/leaf时，Master会需要先获取从/d1、/d1/d2到/d1/d2/.../dn的读锁，然后再根据操作的类型获取 /d1/d2/.../dn/lead的读锁或写锁。**获取父目录的读锁是为了避免父目录在此次操作执行的过程中被重命名或删除。**

由于大量的读写锁可能会造成较高的内存占用，这些锁会在实际需要时才进行创建，并在不再需要时被销毁。此外，所有的锁获取操作也会按照一个相同的顺序进行，以避免发生死锁：锁首先按Namespace树的层级排列，同一层级内则以路径名字典序排列(**"OSTEP"中chapter32讲述的，使用一定的锁的顺序来避免死锁**)。

### Lease管理[2]

GFS使用租约（lease）机制来保持多个副本间变更顺序的一致性。在客户端对某个Chunk做出变更时，会把该Chunk的Lease交给某个Replica，使其成为Primary：Primary 会负责为这些变更安排一个执行顺序，然后其他Replica便按照相同的顺序执行这些修改。

设计租约机制的目的是为了最小化Master节点的管理负担。Chunk Lease在初始时会有60秒的超时时间。在未超时前，Primary可以向Master申请延长Chunk Lease的时间；必要时Master也可以直接撤回已分配的Chunk Lease。

# Read and Write

在写文件的时候会涉及到Lease和Version的问题。

## Read

1. Client将文件名 + Offset转为文件名 + Chunk Index，向Master发起请求

2. Master在元数据中查询对应Chunk所在的Chunk Handle + Chunk Locations并返回给Client

3. Client将Master返回给它的信息**缓存**起来，用文件名 + Chunk Index作为 key

4. Client会选择网络上最近的Chunk Server通信，并通过 Chunk Handle + Chunk Locations 来读取数据

## Write

<div align="center"> 
<img src="./imgs/GFS-write-control.png" width = "40%"/>
<br>
    <div style="color:orange; border-bottom: 1px solid #d9d9d9;
    display: inline-block;
    color: #999;
    padding: 2px;">Fig 2. Write Control and Data Flow[1]</div>
</div>

写文件可分为 7 步：

1. Client向Master询问哪个Chunk Server持有该Chunk的当前租约，以及其他副本的位置。如果没有人有租约，则Master将租约授予它选择的Replica。

2. Master回复谁是Primary和Secondary Replica的位置。Client缓存此数据以备将来更改。仅当Primary变得不可访问或它不再持有租约时，Client才需要再次联系Master。

3. Client将数据推送到所有Replica。Client可以按任何顺序执行此操作。每个 Chunk Server都会将数据存储在内部LRU缓冲区缓存中，直到数据被使用或老化。通过将数据流与控制流分离，我们可以通过基于网络拓扑调度不同数据流来提高性能，而不管哪个Chunk Server是Primary。

4. 一旦Client确认每个Chunk Server都收到数据，Client向Primary发送写请求，Primary可能会收到多个连续的写请求，会先将这些操作的顺序写入本地(以此来避免并发问题，顺序对于其他Replica来说是一致的)。

5. Primary做完写请求后，将写请求和顺序转发给所有的Secondary，让他们以同样的顺序写数据。

6. Secondary完成后应答Primary写请求是否成功。

7. Primary应答Client所有的流程是成功还是失败。如果出现失败，Client会重试，但在重试整个写之前，会先重复步骤 3-7

## Atomic Record Appends

文件追加的操作与写文件的操作非常像。

1. Client将数据推送到每个Replica，然后将写请求发往Primary。
2. Primary首先判断将数据追加到Chunk后是否会令CHunk的大小超过上限。如果是，那么Primary会将当前Chunk填充至其大小达到上限，并通知其他Replica执行相同的操作，再响应客户端，通知其应在下一个Chunk上重试该操作。
3. 如果数据能够被放入到当前Chunk中，那么Primary会把数据追加到Chunk中，拿到追加成功返回的偏移量，然后通知其他Replica将数据写入到相同位置中。
最后Primary把偏移量返回给Client。

**NOTE！！！**

> GFS只确保数据会以一个原子的整体被追加到文件中**至少一次**。如果一个Replica追加不成功，那么会重试，此时可能会发生在一个Replica中出现两次该文件，但是不论如何，他们的偏移量都是一样的。
> 
> 类似于这样的(从上到下看)：
>
> | Replica 1(primary)| Replica 2(secondary) | Replica 3(secondary) | state |
> | :---:        |    :----:   |          :---: | :---:|
> | A      | A       | A   |- |
> | B   | Failed        | B      |write B |
> | C   | C        | C      |write C |
> | B   | B        | B      |try write B again|


# Consistency

<div align="center"> 
<img src="./imgs/GFS-consistency.png" width = "30%"/>
<br>
    <div style="color:orange; border-bottom: 1px solid #d9d9d9;
    display: inline-block;
    color: #999;
    padding: 2px;">Fig 3. File Region State After Mutation[1]</div>
</div>

GFS是弱一致性的模型。它并不保证一个 chunk 的所有副本是相同的。从文件追加中就可以看出来。

Inconsistent：客户端读取不同的Replica时可能会读取到不同的内容，那这部分文件是不一致的。

Consistent：所有客户端无论读取哪个Replica都会读取到相同的内容，那这部分文件就是一致的。

Defined：所有客户端都能看到上一次修改的完整内容，且这部分文件是一致的，那么我们说这部分文件是确定的。

# materials

check FAQ of MIT6.824 2022 Lecture 3[3]


# Reference

[1] Goole File System. Sanjay Ghemawat, Howard Gobioff, and Shun-Tak Leung. Proceedings of the 19th ACM Symposium on Operating Systems Principles, ACM, Bolton Landing, NY (2003), pp. 20-43

[2] [SOSP'03 The Google File System, from zhihu](https://zhuanlan.zhihu.com/p/87314382)

[3] [MIT 6.824 2022, distributed system](https://pdos.csail.mit.edu/6.824/papers/gfs-faq.txt)