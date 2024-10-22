# DRust Runtime System

runtime system includes:

* a runtime library
  * linked to each application and launched on each server
* a cluster-wise global controller

## 1 Application-Integrated Runtime

runtime library includes:

* a communication layer
  * support inter-server coordination and data transfer
  * control plane
    * 双通道
  * data plane
    * 单通道
* a heap allocator
  * manage the heap partition and the read-only cache
  * 会优先在本地分配
  * 如果本地不够了，会通过global controller来找到空闲的server，借助communication layer来在远程server上分配，释放的时候不需要gc，可以直接联系server释放。关于cache，会把它当作heap的一部分来处理
* a thread scheduler
  * launch and schedule application threads
  * 支持单机内的线程调度和分布式的线程调度，运行在Umode
  * 把新线程看作一个闭包，包含一个函数指针和初始化参数，和gc合作，为线程分配stack，然后执行闭包
  * 非抢占式上下文切换，上下文切换用函数调用处理
  * migrate可以直接把函数指针，栈，寄存器给目标server，因为线程的栈空间都是全局不重合地址

## 2 Global Controller

维护了一个global table，记录线程的位置，如果出现线程迁移，需要更新这个表。

定期访问每个server，记录cpu和memory。

内存分配 线程迁移都和它有关。

如果说在一个服务器上启的线程太多了，那就会执行线程迁移。

cpu或者memory大于90%的话，就会进行migrate。

## 3 Fault Tolerance

对于heap有个backup server，每次mutable borrow之后，都会记录，然后等到有人使用的时候，像backup写batch数据
