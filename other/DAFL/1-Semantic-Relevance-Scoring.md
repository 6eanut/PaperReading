# Semantic Relevance Scoring

* Input: selectively coverage feedback
* Output: semantic relevance score

离目标点最远的节点是1分,离目标点最近的节点是最高分

一个种子的分,是其所覆盖的节点的分的总和

分数越高,优先级越高

---

分数由两个因素影响

* 用某个seed执行的program，其所exercise的node更多，分更高
* 用某个seed执行的program，其所exercise的node，更接近t所在的node，分更高

某个seed的分数，由用该seed执行的program所exercise的node的分的总和得出

某个node的分数，由其在DUG中距离t所在的node的距离影响

* c1和c2的距离是两者在DUG中的最短距离
* 离t所在的node最远的node，分最低；最近的node，分最高
