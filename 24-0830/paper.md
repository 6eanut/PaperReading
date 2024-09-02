# Execution Migration in a Heterogeneous-ISA Chip Multiprocessor

**Heterogeneous CMP & Homogeneous CMP**

* CMP是 Chip MultiProcessor的简称，就是一个系统上有多个处理器
* 异构CMP是指系统上的多个处理器架构、性能等不同；同构CMP是指系统上的多个处理器拥有相同的架构和功能
* Single-ISA Heterogeneous CMP指的是各个处理器采用的是相同的ISA，但是性能、功能等不同

**Migration**

1. reschedule the process on another core
2. change page table mappings to facilitate access to the code for the migrated-to core
3. perform binary translation until a program transformation point
4. transform program state for execution on the new architecture

**Memory Image Consistency**

* *Global Data Consistency:* add support for small data sections to the GCC ARM back-end
* *Code Section Consistency:* the order of function definitions in memory and the size of each functionbe must be identical
* *Heap Consistency:* the same implementation of malloc be used for all ISAs
* Stack Consistency
