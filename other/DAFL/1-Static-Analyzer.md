# Static Analyzer

* Input: a program with an annotated target location
* Output: Def-Use Graph and a set of relevant functions with respect to the target point
* Exe: run an inter-procedural static
* Purpose: identify all the statements that the target location is data-dependent on

DUG的构建：对输入程序执行跨过程的静态分析，以识别所有与目标位置数据依赖的语句。这包括追踪变量从定义到使用的路径

数据依赖收集：基于DUG，从目标位置开始反向遍历程序，收集所有依赖于该目标位置的数据流节点

优化：因为DUG不包含条件语句，所以对程序进行切片，然后收集DUG

---

输入：program P和target location t

输出：Def-Use Graph和Relevant Functions

> P由CFG表示
>
> CFG由C和->表示，C是node的集合，->是node之间的控制流边的集合
>
> t由C中的一个node表示

DUG的生成

* DUG由C和~>表示，C是CFG中的C，~>是数据依赖边
* c1~>c2的定义是：
  * c1->+c2 &
  * x is a variable in P &
  * x is defined at c1 &
  * x is used at c2, but not for pointer deference &
  * x is not defined at any node between c1 and c2
* 以上步骤完成后，需要对DUG进行剪枝：
  * 从t所在的node开始反向遍历DUG，留下那些和t相关的node，具体做法是：
    * DUG剪枝后得到G，由Ct和~>t表示
    * Ct={c是C中的一个元素 &
    * c~>+t}
    * ~>t={两个node都是Ct中的元素 &
    * c1~>c2}
* 至此，就得到了最终的DUG

Relevant Functions的生成

* 包含c节点的函数被称为相关函数
* c节点是DUG中Ct的一个元素
