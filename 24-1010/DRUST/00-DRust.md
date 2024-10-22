# **阅读部分：**

## **阅读论文的标题、摘要、简介**

### 题目

* 啥是分布式？

  * 资源分散：计算资源(CPU)和存储资源(硬盘)分布在多个物理节点
  * 并行处理：任务可以在多个节点上并行执行
  * 冗余和容错：通过在多个节点上冗余存储数据，增加系统的可靠性和容错能力，当某个节点出现故障时，其他节点可以继续提供服务
  * 负载均衡：动态分配任务和资源，避免某个节点过载
  * 数据共享：节点之间可以共享数据
  * 动态扩展：添加/移除节点
  * 异构型：支持不同的硬件和操作系统
* distributed shared memory(分布式共享内存)

  * 允许在多个计算节点之间共享内存资源，使得分布在不同的物理位置的计算机能够像访问本地内存一样访问远程内存
  * 粒度、透明性、效率
    * granularity
      * 指DSM中共享内存的基本单位，粒度大的话就是整个页或者大块内存，小的话就是单个字或者字节。
      * 小粒度可以提供更细粒度的控制，但是管理开销大；一致性管理机制困难，需要更多的数据同步；大粒度会提高性能，但是会导致不必要的数据共享
    * transparency
      * 指用户在访问远程内存时的体验程度。高透明性意味着用户不需要意识到内存是分布式的
      * 位置透明性，用户不需要知道数据存储在哪个节点；访问透明性，可以用标准内存访问方式来访问；迁移透明性，数据在节点间移动，不影响用户访问
      * 总之就是简化程序开发、降低开发者负担
    * efficiency
      * dsm在内存访问和数据共享时的性能
      * 网络延迟；带宽利用率；一致性协议
* languaged-guided

  * 利用某种语言来指导系统的行为或决策
* DRust: Language-Guided Distributed Shared Memory with Fine Granularity, Full Transparency, and Ultra Efficiency

  * 文章介绍了DRust这么一个模型？这个模型是一种利用语言来引导的DSM，这具有良好的粒度、完全透明和超高效率

### 摘要

在DSM系统里面，因为需要实现服务器之间的内存一致性而导致昂贵的同步开销，这让DSM系统虽然是一个很有效的概念但是却迟迟没有投入使用。

> memory coherence：内存一致性主要被考虑在多处理器和分布式系统内，它确保在多个处理器访问共享内存时，所有处理器对同一共享数据的视图保持一致。当一个处理器修改共享内存中的数据时，其他处理器应该能够在合理的时间内看到这个修改。除此之外，还要考虑缓存一致性，每个处理器都有自己的缓存，如果缓存修改了数据，其他处理器也需要直到这个变化

作者们观察到，具有ownership特点的编程语言可以自动地限制读写顺序，如果运行时能够利用这种特点，那么像rust这类语言就可以实现简化内存一致性的目的。

本文写了DRust(基于Rust实现的DSM)的设计和实现，比目前最好的GAM和Grappa吞吐量都要好，而且如果服务器数量变多了，DRust也可以很好的适应。

### 简介

DSM在分布式计算系统早期很火。有很多很厉害的成果(看引用)。DSM特别适合并行计算，因为它能用多处理器，并且更重要的是，它能简化应用了统一连续内存视图的分布式应用程序的开发。

因为普遍存在网络速度低的问题，DSM具有严重的性能瓶颈，这导致其热度下降。随着硬件和网络技术的发展，解决了上面的问题。但是目前性能还不行，相比于单个机器，其扩展性和性能都不行。主要是因为需要在服务器之间实现内存一致性而导致的昂贵的同步开销。

**最先进的技术。**现存的DSM的设计是这样的，如果一个node对block有写权限，那这个block就只会在这个node上；如果只有读权限，那么这个block可以给多个node复制一份，只用来读。

如果有个node想访问block，dsm会检查这个block的状态，让这个块在其他node上的副本失效，然后把这个block传给发出请求的node。这整个步骤，需要很大的网络延迟。尽管利用RDMA，但是性能也比只用单个的node要低得多。

一种解决方法是实现一个高级别的协议，来保证nodes的独占访问。比如apache spark实现了RDD这种不可变的数据结构来实现分布式访问。但是RDD的粒度太大。

虽然粒度变大能提高性能，但是通用性降低了。比如Spark是为批处理数据定制的，并不支持对象级的访问(常见于分布式应用)，像社交网络这种会有不同类型大小的对象就不能使用。

**观察。**现在的DSM都没有利用程序的语义信息。很多并发程序都是设计成了SWMR（Single Writer Multiple Reader），如果说能够利用这个信息，那么访问block前可能不需要去检查状态。现在的问题是如何把这种信息交给DSM来看到并使用。

可以通过暴漏API来让开发人员使用，以指定只有单个编写者才能访问的程序区域。但是这样的实现比较复杂并且容易出错，还需要写很多代码。

作者发现SWMR和ownership可以结合，并且像rust这种具有ownership的编程语言已经在系统社区里面被广泛使用，对于实现可靠安全的低级系统很有效。

rust的ownership特性的实现就包含了SWMR的实现。ownership就是每个value在执行过程中都有一个唯一地variable作为其owner。尽管一个value可以有很多引用，但是只有owner和可变引用能修改value，而且可变引用只能有一个。

如果用rust来开发dsm，那么SWMR也会自然而然地被实现。得益于编译器，代码不需要重写。利用SWMR信息，DSM的数据访问过程被简化了。对于写访问，ownership保证了独占访问，block可以直接给发出请求的node，不需要让block在其他node上失效。对于读访问，得益于编译器，block可以快速复制在每个node上。

**总结。**本文提出了drust，通过利用rust的ownership来实现SWMR语义的利用，实现了对象级别的并发访问。drust能够在不重写代码的情况下，让单机rust程序在DSM上跑。drust的设计主要有两个难点。

* 如何正确高效地管理内存。
  * rust的ownership是为单机环境设计的，对象的内存地址在生成后是不会变得。如果在分布式里面，对象可能会被迁移、复制到不同的机器，内存的管理是很重要的。否则会出现dangling pointer，可能会破坏内存一致性。
  * 解决方法是实现了一个跨多服务器的全局堆。在这个堆中的对象可以被任何服务器访问。drust改写了rust的内存管理结构，以此分配全局堆中的对象。drust还实现了缓存一致性协议，实现快速读。
  * 这个协议允许多个reader拷贝对象并且缓存，但是不允许他们修改。如果有writer，drust会把全局堆上的对象ownership借用给writer。
* 如何实现编程透明性。
  * rust的标准库和程序是为单机环境设计的。如果在分布式里面，会无法处理资源。运行在a服务器上的rust程序，不能在b服务器上产生一个线程，也不能同步a和b之间的线程。
  * 解决方法是重构rust标准库，threading， communication channels，shared-state locks。适配后的库不仅兼容单机环境，还支持分布式调度器，决定线程在哪运行，并且能跨服务器进行同步。

通过重写内存管理和重构rust库，可以让rust程序在一个server上启动，然后把线程衍生到其他服务器上。每个server上都有一个runtime来管理内存和调度。还有一个global controller来决定全局内存分配和线程创建，这个控制器是和application一起启动的。global controller会和每个runtime沟通，来获得使用资源信息等。

drust在八节点的集群上测试了四个应用，性能比GAM和Grapp好。比单机环境下的rust性能小一点点就。

## **阅读章节和子章节的标题**

[树状图](00-DRust/tree.png)

### Background in Ownership

编程语言越来越关注安全的内存管理和数据共享，内存抽象级别和管理效率需要权衡。rust在权衡方面做的很好。这部分解释为啥rust适合用来实现dsm。

ownership这个model早就有了，用于内存管理语言设计和类型系统。这类语言在编译的时候会进行类型检查以保证内存和线程的安全。ownership模型中最重要的概念是lifetimes和borrowing。

> lifetimes用来控制一个对象的分配和释放。每个对象都有且仅有一个owner，编译器在可以静态跟踪对象的lifetimes，如果owner超出scope，就会释放，不需要GC。
>
> 如果一个程序要访问一个对象，那就得创建这个对象的owner的reference，即borrow这个owner的permission，访问完之后再return给owner。type system允许创建多个不可变引用来读object，如果要写，就只能有一个引用，且是可变的。这能避免data race

ownership模型有四个特点：singular owner，safe borrowing， single writer， multiple reader

后两个不就是SWMR嘛，所以说具有ownership的语言，自然而然就会满足SWMR

rust就是实现了上述ownership的编程语言。文中给了一个rust程序，来说明上述四个特性。

### Motivation

dsm的作用是让分布式编程能够像在单机环境下一样共享内存。核心思想就是通过软件缓存一致性协议来模拟硬件，通过在各个服务器之间发送消息来实现内存状态的同步。但是延迟太大了。

用最好的GAM来跑DataFrame，发现分布式比单机要慢得多。GAM是directory-based，一旦有读和写访问，home node就会跟踪状态变化，并且更新所有copies，这就有网络开销很大

### Design

rust application通过drust abstraction把ownership semantics给了drust runtime system，然后后者管理物理资源。

drust abstraction分为ownership-oriented memory model， standard library， affinity hints

drust runtime system分为communication layer， heap manager， thread scheduler和global controller

## **阅读总结**

## **阅读引用**

# **收获：**

## **实践/理论？**

## **主要贡献？**
