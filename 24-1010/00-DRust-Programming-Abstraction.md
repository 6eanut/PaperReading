# Rust Programming Abstraction

* ownership-oriented memory model
  * drust为每个线程分配一个栈和一个全局共享堆。每个服务器会给线程分配栈并且在全局共享堆里面选一块给该线程。drust重新实现了Box，&和&mut来实现内存管理，实现了内存分配和释放，对象move，一致性保持。
* standard library
  * drust通过更改标准库来实现分布式线程和同步。
* affinity annotation
  * drust还提供了注释供开发者使用。

## 1 Memory Management

* Address Space
  * 服务器们把全局共享堆给划分了，每个人都有一部分；服务器里面的线程独有自己的栈，但是所有服务器中的所有线程栈在地址上是不存在覆盖的。[如图](00-DRust-Programming-Abstraction/Address-Space.png)
* Coherence Protocol
  * call-by-reference
    * 创建线程时，参数只使用reference或者Box指针，这样一旦dereference，object就会还给创建线程的线程
  * read/write
    * read：runtime把object复制到读服务器的local cache
    * write：写服务器server2需要获得object的&mut(re-impl)，这使得原本在server1的heap中的object挪到了属于server2的heap中，从而实现引用了server1中heap该object的读都失效了。
    * 这不仅对于不同server之间生效，对于同一server也有效，这就导致如果是在一个server上执行写，也会给object重新安排空间，这很低效。所以使用了pointer-coloring技术
* Pointer Layout
  * 重写Box & &mut，增加了64位字段，用于读写。读，local copy address；写，owner address。color字段用于提高local write的效率。[如图](00-DRust-Programming-Abstraction/rust-box-&-&mut.png)
* reimpl ownership operation
  * mutable borrow
    * dereference
      * 会先检查global address是不是本地的，如果是，则直接访问；如果不是，则把object复制到本地，然后把object的引用中的GA写成本地的地址，然后让原server把原地址释放
      * ownership的特点3single writer意味着，如果有了&mut，那么原本的owner也无法修改object，这就避免了dangling pointer的出现
    * drop
      * 会把object在写服务器的新地址的内容写回到原owner，就是return
    * 算法[如图](00-DRust-Programming-Abstraction/mutable-borrow.png)
  * immutable borrow
    * dereference

      * 先检查在不在本地。在的话直接读la；不在的话，就从ga复制一份到la。每个&都不仅保留了ga还保留了la。
      * 像上面这样的copy，一个服务器上会把这类copy放在一起，由hashmap管理。每个node都有一个hashmap，记录ga到la的映射，并且记录&到la的数量。所以在copy之前，都会先检查hashmap。
      * 如果说在访问某个object时，在hashmap里面找到了，drust就会更新hashmap里面的那个数量，并且更新&的la。没找到的话，就得copy了。
      * 为啥要有记录数量，是为了drust会周期性扫描，如果说内存压力较大，会把数量为0的删掉
      * 如果说有别的server修改了object，那ga就会改变，相应的这边的就都会失效了
    * 算法[如图](00-DRust-Programming-Abstraction/mutable-borrow.png)
  * owner access without borrow
    * 如果一个可变对象被不可变引用，但是还想改变该对象，那就只能等所有的不可变引用都返回后，再写
  * ownership transfer
    * 只进行指针的copy，不进行值的copy
  * memory deallocation
    * 根据litftime来释放
  * consistency model
    * 每有一个写操作，其后续的读操作得能看到更新后的值
  * optimizing for local writes
    * 如果写操作完成，会让颜色值加1；如果颜色值移除，会把object重新写到一个地方，然后让颜色值归零
    * 如果是读操作，就会看看颜色值变了没，变了的话就说明值被改了；否则直接读
  * writing unsafe code in drust
    * 支持unsafe
    * 提供dalloc dread dwrite 管理全局dsm堆上的数据

## 2 Adapting Rust Standard Libraries

* threading -> distributed computation
  * std::thread
  * 重写spawn。编译时捕获线程体作为闭包，并转发给运行时。运行时根据每个服务器的负载启动线程。
  * ownership transfer会在子线程的创建和结束时发生
* inter-thread channel -> communication
  * std::sync::mpsc
  * 网络消息队列
* reference-counted pointers -> ownership sharing
  * std::sync::Rc
  * std::sync::Arc
  * 分别处理单线程和多线程中的reference counted
* shared-state locks -> concurrency control
  * std::sync::Mutex
  * std::sync::atomic
  * 在全局堆上分配真实值，并且只存储box pointer以原子类型

这一款看的不是很明白，主要原因是rust线程部分，还没学

## 3 Affinity Annotations

开发者可以通过annotation提供数据语义，以此提高性能。这对于充分利用以指针查询方法访问面向对象的数据结构的数据中心应用很有用。

memcached是有一个链表，然后访问的时候需要对指针解引用以及查询，很麻烦。如果说能把这些指针都放在一个机器上，那么效果会好很多。
