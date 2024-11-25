# GAM

## 摘要

背景：高性能网络互联技术提高了节点间通信的速度，为DSM实现节点间缓存一致性提供了机会

做了什么：提出了GAM

* 通过RDMA提供directory-based缓存一致性协议
* 管理节点上的空闲内存，提供统一的内存模型，提供API用于内存操作
* 为了将写操作从关键执行路径中移除，GAM允许写操作和后面的读写操作进行重排序，实现部分存储顺序PSO内存一致性
* 轻量级日志以提供故障恢复能力
* 在GAM上构建了事务引擎和分布式哈希表

测试：测试GAM在不同负载下的读写锁性能，对事务引擎和分布式哈希表进行测试

## 简介

shared-memory和shared-nothing对比

* 前者，提供统一全局内存抽象，便于开发人员；负载均衡
* 后者，键值存储系统，易于扩展

现有DSM提供缓存来缓冲远程内存访问，以减少网络延迟；但是为了维持缓存数据一致性，需要同步原语来更新缓存中的数据，开销过大；而且这需要程序员来完成

现在RDMA技术几乎接近本地访存、甚至比NUMA的吞吐量都大，但是还是需要程序员来完成同步原语

一种方法是不用缓存，直接向数据所在节点进行读写，但是这样延迟太太大了

所以本文提出了GAM，保留了缓存功能，利用RDMA来实现高效的缓存一致性协议

## 系统设计

### a-内存模型和API

采用PGAS(Partioned Global Address Space)内存模型，逻辑上统一的地址空间+机器间通过RDMA互连+每个节点管理自己的那块内存

API分为访问全局内存空间操作和同步操作两类，前者是malloc/free和read/write，后者是atomic、mfence、rlock/wlock、tryrlock/trywlock、unlock

### b-缓存一致性协议

虽然RDMA很厉害，但是本地访问和远程访问之间的延迟还是差10倍。大多数应用都有数据局部性的特点，利用这点可以利用层次化内存架构，来减少访问较低层次的内存的需求。GAM在全局内存之上，增加了一层DRAM的缓存

这个缓存层不采用窥探协议是因为，RDMA网络不适用，所以采用了基于目录的协议，即维护一个目录，记录数据块的信息，当需要同步缓存时，点对点通信，而非窥探式的广播

GAM有三层缓存层，最高层是由硬件实现的NUMA节点内的基于窥探的协议，第二层是由硬件实现的NUMA节点间的基于目录的协议，最底层是由软件实现的本文提出的协议，接下来说下这层的协议

对于每块数据而言，节点们可以被分为五种类型，home数据的物理内存所在节点，remote非home节点，request请求对数据进行访问的节点，sharing拥有读权限的节点，owner拥有写权限的节点

最初，数据在home上，home既是sharing也是owner，如果有remote变成了request，那么home就会把权限给过去，该remote就会被提升为sharing或者owner。对于一个数据块而言，可以同时有多个sharing，但是只能有一个owner，且sharing和owner不能同时存在。这种缓存效果可以利用数据局部性

缓存的粒度，默认是512bytes，比硬件的要大，这样能减少小数据包传输的开销(因为每次传输有固定开销)，并且这个数字是在平衡了网络带宽和延迟之间做的选择

每个缓存行都有状态，对于home，有unshared、shared、dirty三种状态，对于remote，有invalid、shared、dirty三种状态，状态间的转换不是原子性的，所以又引入了中间的过渡状态

### c-读

读操作分为local和remote两种，前者指的是home作为request，后者指的是remote作为request。

对于local，如果Cache line是shared/unshared，那么直接读；如果是dirty，需要查Directory entry，然后找到owner，owner要把本地copy发给home，并且把Cache line从dirty改为shared，home收到copy后会更新memory line，并且更新directory entry到shared

> **问题**
>
> 为啥sharing-nothing比sharing-memory容错性好，后者如果单个节点故障，会发生什么？
>
> 为啥RDMA的吞吐量可以，延迟缺不怎么行？
>
> 啥是基于窥探的协议，snoop-based cache coherence protocol
