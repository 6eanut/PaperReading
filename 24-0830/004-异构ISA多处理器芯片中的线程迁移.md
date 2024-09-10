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

**Migration Process**

* *Stack Transformer*: creates the register state for the migrated-to core; fixes all the return addresses on the stack; moves the values of local variables in open function activations to the right stack offsets
* *Optimizations that Interfere with Variable Location:* the optimizations can be converted from RTL passes to tree passes；the optimization passes can be modified to avoid applying the optimization across call sites
* *Operation of the Stack Transformer:* ~~The first pass goes from innermost frame (the frame in which execution was stopped) to outermost frame and finds values for caller-saved spill locations and simple stack-bound variables; The second pass works in the reverse direction—from outermost frame back to innermost framefinding values for callee-saved spill locations and determining final register state~~**看不懂**

**Binary Translation**

* Translation Block Chaining: The translation engine, before translating the next block of instructions, checks if the block is already available in the code cache. If it is available, it links the end of the previous block to the beginning of the next block, with a direct branch instruction
* Multiple-Entry Multiple-Exit (MEME) translation block chaining: allowing translation block chaining from any instruction in the middle of a TB to any instruction in another TB
* ISA-Specific Challenges: Register Allocation; Condition Codes and Predicated Instructions; Immediate Instructions; System Calls

**Experimental Methodology**

说了下测试的方法

**感悟**

第一次读论文，首先16页感觉还挺长的。内容方面感觉是高开低走 最开始的intro讲的很厉害，后面看的时候会发现加了很多限制，不过这也正常。

首先先是略读，也就是直接看中文翻译，大致明白是干啥的，搞清楚一些关键概念，然后再去细读，像最后的result部分就没有怎么看仔细。
