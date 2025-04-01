# SyzDirect

Linux内核DGF模糊测试工具，给定某个位置，对该位置进行stress-test。

## 难点

1. 如何确定触发该位置的syscall；
2. 如何确定触发该位置的syscall的参数值。

## 解决方案

![1743405447308](image/note/1743405447308.png)

* Entry Point Identification：从target入手，得到调用链，分析用到的资源，结合这些信息，筛选可能的目标syscall。即kernel function和syscall的对应，用到了syzlang；
* Syscall Dependency Inference：得到和目标syscall相依赖的syscall，从资源的创建和使用方面来入手，这个在syzlang里面提供了；
* Syscall Argument Refinement：syscall确定后，就会根据条件语句确定系统调用的参数，这个也要结合syzlang；
* Directed Kernel Fuzzing：得到上述信息后，会用来指导种子变异，定制了变异规则；

## 细节

### Entry Point Identification

两个观察：

1. 用syzlang写的syscall变体，具体指出了要操作的资源以及如何操作；
2. syscall进入内核之后，会进行dispatch，这一过程取决于要操作的资源和操作的方式。

过程：

1. 将dispatch后的第一个函数当作anchor函数，将其与syscall变体匹配；其他函数需要追溯到调用该函数的anchor函数，然后与anchor函数匹配的syscall变体匹配；
   1. 将dispatch后的第一个函数当作anchor函数：分析syscall变体，得到参数的具体值和类型；分析核函数，找到所有switch语句，对比switch的变量和参数的类型，匹配之后，将case是参数具体指的函数当作anchor函数；
   2. 参数的指定：追溯syscall变体要用到资源是哪个syscall变体创建，分析创建者的常量参数，将常量参数与该资源匹配；人工查找注册函数，进而找到创建资源函数，然后找到与资源相关的赋值语句，比如常量或虚表；
2. 

给定一个target code location，先找到可以到达该target的primitive syscalls，然后结合参数对应到syscall variants。

---

这一部分的目标是找到能够触发目标代码位置(target code location)的系统调用变体(syscall variants)，也叫做入口系统调用(entry syscalls)。为了实现这个目的，需要将内核函数(kernel functions)和系统调用变体对应起来。因为相比于原始系统调用(primitive syscalls)，系统调用变体的粒度更细，即记录了在特定资源上的特定操作，故而将内核函数与系统调用变体匹配，而不是原始系统调用。具体的匹配方法是用相同的方式对内核函数和系统调用变体的操作和资源建模。

对于内核函数而言，并不是所有的内核函数都显式的给出资源和操作，所以没法直接获得。但是，当一个系统调用被执行时，在内核里，会进行函数分发(function dispatch)，即根据系统调用参数来决定要处理的资源和要执行的操作。所以可以通过观察函数分发的过程，来分析内核函数要操作的资源以及具体操作。这里把函数分发过程后的第一个函数当作锚函数(anchor function)，首先对锚函数进行建模并匹配系统调用变体，除了锚函数之外的函数会被对应到可以到达该函数的锚函数，并与锚函数的系统调用变体进行匹配。
