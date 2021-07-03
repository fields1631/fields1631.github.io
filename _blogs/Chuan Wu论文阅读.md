---
title: "Chuan Wu论文阅读"
collection: blogs
type: "blogs"
permalink: /blogs/Chuan Wu论文阅读
date: 2021-06-01
---

## DAPPLE: A pipelined data parallel approach for training large models

Fan, S., Rong, Y., Meng, C., Cao, Z., Wang, S., Zheng, Z., Wu, C., Long, G., Yang, J., Xia, L., Diao, L., Liu, X., & Lin, W. (2021). DAPPLE: A pipelined data parallel approach for training large models. *Proceedings of the ACM SIGPLAN Symposium on Principles and Practice of Parallel Programming, PPOPP*, 431–445. https://doi.org/10.1145/3437801.3441593

介绍DAPPLE，是使用新的并行策略的框架，同时使用了data parallel与pipeline parallel，并且提出了一种调度算法，相比recomputation的调度算法，不需要更多的计算量。调度算法使用pipeline parallel，其创新之处在于early backward，减少激活函数输出等临时计算结果在内存中储存的时间，以此减小平均内存使用。目前的疑问有：

## A generic communication scheduler for distributed DNN training acceleration

Peng, Y., Zhu, Y., Chen, Y., Bao, Y., Yi, B., Lan, C., Wu, C., & Guo, C. (2019). A generic communication scheduler for distributed DNN training acceleration. *SOSP 2019 - Proceedings of the 27th ACM Symposium on Operating Systems Principles*, 16–29. https://doi.org/10.1145/3341301.3359642

介绍PACE，是为分布式DNN规划all-reduce通信算子与dnn算子的算法。此类系统的架构（应该有offline与online之分）是使用profiler提取出DAG，DAG中每个节点是一个computation operator或一个communication operator，DAG中每个边表示节点之间的依赖关系，然后经过tensor的融合以减少communication operator的个数，并且充分利用带宽，最后规划通信策略，再次运行。

![截屏2021-05-26 08.03.37](/Users/fields/Pictures/截屏/截屏2021-05-26 08.03.37.png)

规划通信策略用到了非FIFO的通信机制。目前通信库主要是FIFO的，没有抢占时的，而PACE规划之后将多个all-reduce operator进行融合（也即tensor fusion），并且将all-reduce operator分解成粒度更细的（fine-grained）operator，借此实现抢占式的通信。这样的实现是预先规定的，实际上还是利用了FIFO的通信库。**如果能有preemptive的通信库会不会好一些？但是对于DNN的scheduling或许没有太大意义，因为网络结构不会变化，每个all-reduce operator传输的数据量不会变化，因此预先的scheduling是很合理的，没有不预先规划而要临时抢占的需要。但搜索得知应该没有非FIFO的通信库，这是一个缺失。**

在推导规划策略的部分，文章先列出了各个符号的定义，主要值得注意的有，时间上存在time slot，即以一小段时间作为处理的对象。时间越短，粒度越高。每个time slot的决策为$\pi(x)$，在执行某个任务时$\pi(x)$为1，否则为0，这可能是比较通用的定义和处理方式。

文章将优化的目标简化为从头到尾计算时间最长的路径以及all-reduce operator有关的时间。在其他模型中，如决策树、softmax、knn中是否也存在。

图卷积神经网络的训练（data parallel）也是一个DAG，没有一定要有环的存在，因此不算是非常合适的主题。

在all-reduce operator的内部存在不同阶段，如将一小部分梯度循环地进行传播，然后进行混合，再准备传播。在这个过程中，在混合的时期能够利用带宽传输其他的all-reduce operator的一小部分梯度，这是tensor fusion中可以处理的。tensor fusion需要混合许多tensor，但是它们的传输和混合并没有仔细地进行规划过。如tensor1和tensor2，一种方式，也是我猜测tensor fusion使用的方式，是先传输tensor1，然后传输tensor2，然后进行混合，然后继续传输。但也可以传输tensor1，在tensor1混合的同时传输tensor2，在。

需要解决的问题是：all-reduce operator的传输与混合的时间比例，以及tensor fusion的具体实现。

目前的想法是：Currently I am considering the following design. When all chunks of original tensor *t* is broadcasted in the overlapped broadcast stage of the fused tensor, the computation operator whose original precedent AllReduce operator exactly handles the synchronization of tensor *t*, can start working immediately instead of waiting for the entire broadcast stage of the fused tensor. This design contains not only the overlap of reduce and broadcast stage of fused AllReduce operator, but also the overlap of fused AllReduce operator and computation operator in distributed DNN training.（来自对吴老师回复的邮件）

## Near-Optimal Topology-adaptive Parameter Synchronization in Distributed DNN Training

Zhang, Z., Wu, C., & Li, Z. (2021). Near-Optimal Topology-adaptive Parameter Synchronization in Distributed DNN Training. *Infocom*.

这篇介绍了参数同步的方法并且提出了新的拓扑结构来优化参数同步的带宽与速度。

首先了解到已有的分布式训练方法是参数服务器（parameter server）以及allreduce架构。其中ring-allreduce被证明是对带宽最优的选择。但是其最有只存在于homogenerous的假设下，如果各个worker的带宽情况不同，则其不是最优的。

值得记住的是，文章介绍GPU之间的连接有PCIe的方式，GPU之间连接有多方式（TODO：整理一下连接方式并在这里列举）。

通过将每个chunk无限划分并且pipeline地传输，可以推导出一棵reduce tree或一棵broadcast tree的等效带宽。等效带宽取决于树的结点之间连接的最小的带宽。为了最大程度利用带宽，文章将chunk的大小与每棵树的等效带宽匹配，并且提出传输时间的优化目标

