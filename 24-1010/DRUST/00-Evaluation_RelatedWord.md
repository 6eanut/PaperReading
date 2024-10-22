# Evaluation

* setup：硬件、OS、禁用超线程..
* methodology：DRust、GAM、Grappa进行比较，说了下在GAM和Grappa上的工作

## 1 Applications

选了四个应用，考虑到资源需求和适用范围

* DataFrame
  * in-memory data analyticcs framework
  * tbox spawn_to
* SocialNet
  * twitter-like web service
  * pass only references
* GEMM
  * matrix multiplication
  * shared memory
* KV Store
  * in-memory key-value cache engine
  * shared memory

## 2 Scaling Performance

1 单机环境，测量吞吐量

2 增加server数量，测量吞吐量

> GAM和Grappa无法处理servers之间的负载平衡，所以是手动平均分配

3 按照假设，吞吐量应该和sever数量线性变化，但是由于一致性维护，GAM和Grappa很难达到，DRust几乎实现了

dataframe频繁读写，共享内存，而且用了affinity annotations

socialnet需要数据序列化和解序列化，但是因为是reference传值，所以避免了这一点

gemm需要频繁计算，不频繁读共享内存，drust第一次把数据复制到本地，后面直接读本地，更快

kv store使用独占访问每个worker，drust因为单通道数据传输和平衡负载更厉害

## 3 Drill-Down Experiments

affinity annotations：对比了pointer和thread的好处

runtime dereference check：检查了drust里面box和原本rust里面box的差别

thread migration latency：挺快的

cost of cache coherence：测量一致性内存和跨服务器内存访问成本

# Related Word

software dsm system: munin, treadmarks, rdma

disaggregated and remote memory: 给host提供一个大的内存池，通过快速的数据中心网络来连接，应用通过os或者runtime来访问，但是不支持缓存一致性

distributed programming abstractions：munin，x10，upc，ray，nu

hardware-accelerated dsm：硬件来加速
