# Virtualize-compute-clusters
简要介绍如何构建一台微型“超算”。当然，真的超算要复杂很多，因为涉及到多用户，多任务，队列管理和存储管理等等一系列问题，还涉及到电源，冷却以及网络连接等等。在没有排到大型机的情况下，还是可以应付一些简单的小规模的并列运算。
-----------------------------------------------------------------
用pc构建DIY计算集群
目录
/构建计算集群
0. |-- /0前言
|-- /1理论
|-- /2结构
|-- /3操作系统和软件环境
|-- /4两台PC的集群
|-- /5材料科学/量子化学领域软件

1.理论
======
#1.1并行计算<br>
##并行计算（parallel computing）是指在并行机上，把一个应用分解成多个子任务，分配给不同的处理器，各个处理器之间协同，并行执行子任务，目的是加快计算速度。
由此需要

1，硬件支持，需要并行机，多核，单机的话，单个任务在一个核心上执行和多核上执行速度也不一样，`多机就需要网络连接`。

2，计算的问题可以并行，如果计算的问题是流水线，互相关联度很高，完全没法并行，那么就没有必要用并行计算或者超算。

3，需要进行相应的编程优化。

那么什么样的问题需要超算呢，大规模的科学和工程计算，比如天气预报，需要24小时完成48小时的数值模拟，至少需要635万个网格点，内存大于1TB，计算性能要求高达25万亿次／s。

------
1.2MPI消息传递接口<br>
-------
MPI（message Passing interface）是全世界联合建立的消息传递编程标准，目的是用消息传递提供一个高效可扩展，统一的编程环境，是目前最为通用的并行编程方式，也是主要应用的。MPI有多种"实现"，包括mpich1/mpich2、openmpi、lam-mpi等，每种mpi的实现需要配合相应的编译器，才能发挥作用。

-------

c语言mpi hello world：
-------
```C
#include <mpi.h>
#include <stdio.h>

int
main(int argc, char *argv[])
{
    int rank, size;
    MPI_Init(&argc, &argv);
    MPI_Comm_rank(MPI_COMM_WORLD, &rank);
    MPI_Comm_size(MPI_COMM_WORLD, &size);
    printf("Hello, World.  I am %d of %d\n", rank, size);
    MPI_Finalize();
    return 0;
}
```

1.4并行算法
-----
任何并行算法的设计都是基于某一特定的并行计算模型，而并行计算模型是从各种具体的并行机中抽象出来的，它能在一定程度上反映出具体并行机的属性，又可使算法研究不再局限于某一种具体的并行机。并行算法可从不同的角度分类成数值计算的和非数值计算的并行算法;同步的、异步的和分布式的并行算法;共享存储的和分布存储的并行算法;确定的和随机的并行算法等等。举个例子，单个进程的快速排序<br>
```pseudocode
输入 : 无序序列(Aq ，⋯ ，Ar)
输出 : 有序序列(Aq ，⋯ ，Ar)
Procedure QUICKSORT(A, q, r)
Begin
if q < r then
x = A[q]
s = q
for i = q + 1 to r do
if A[i] ≤ x then
s = s + 1
swap(A[s], A[i])
endif
endfor
swap(A[q], A[s])
QUICKSORT(A, q, s)
QUICKSORT(A, s+1, r)
endif
End
```

------
然后进行并行化之后是这个样子，基于二叉树的并行选主元的PRAM_CRCW 模型上的快排序算法。执行快排序可以看成是构造一棵二叉树，其中主元是根，小于等于主元的元素处于左子树，大于主元的元素处于右子树。算法首先从选第一个主元开始，它划分原序列为两个子序列;然后相继子序列中主元的选取则可并行执行。当这样的二叉树造好后，采用中序遍历的方法就可得到一个有序序列。

------
```pseudocode
输入: 序列 ( A1 ，⋯ ， An ) 和n个处理器
输出: 供快排序用的一棵二叉树				
Begin
    (1)for each procesor i do			
        (1.1)root=i
        (1.2) fi =root
        (1.3) LCi = RCi = n + 1					
    endfor
    (2)repeat for each procesor i≠root do 
    if (Ai < Afi) ∨ (Ai = Afi ∧ i < fi) then		
        (2.1)LCfi =i				
        (2.2)if i = LCfi then exit else fi = LCfi endif			
        else
        (2.3)RCfi =i		
        (2.4)if i=RCfi then exit else fi = RCfi endif
        endif
     end repeat					
End
```

-------

2超级计算机的结构<br>
=====
*节点（node）。每个节点由多个处理器构成，可以直接输入输出（I/O）。一般来说登陆节点和计算节点应该分开的。<br>
*互联网络（interconnect network）。连接节点。<br>
*存储（memory）。由多个存储模块构成<br>

------

2.1节点
-----

需要cpu，有缓存，有内存，还需要网络连接，专用的工业超算还需要专门设计节点，比如下图就是cray的节点。<br>
![](https://pic4.zhimg.com/80/v2-bcb98be32d7abc552534caf6bdd4c947_720w.webp) <br>

[Performance Computer, XC Series Supercomputers](http://www.cray.com/products/computing/xc-series?tab=technology "进入网站")

2.2网络拓扑
----

有了节点，就需要对节点进行连接，评价一个网络的准则应该是，对于一定数量的节点，点对点的带宽越高，折半宽度越大，或者网络直径越小，点对点延迟越小，质量就越好。拓扑结构可以分为静态，动态，和宽带连接三类。<br>
a静态连接，就是对于固定数量的节点，单独设计的一套固定的物理连接，至于用什么线连接，都是对用户不可见的，具体取决于厂商或者设计者。连接的拓扑包括阵列（array），环（ring），网格（mesh），网格环（torus），树（tree），超立方体（hypercube），蝶网（butterfly），Benes网等等。详情可以参考其他文献，*《面向拓扑结构的并行算法设计与分析》李晓梅等。现在比较有名的超算一般采用超立方体（hypercube）四维立方体，长这样<br>
![](https://pic4.zhimg.com/80/v2-61a0a029aa6139cdbc357aaebb270a23_720w.webp)  <br>
b动态连接，静态虽好，但是扩展比较困难，如果采用比较简单一维阵列，一个节点挂了，整台都挂了，而且不适合异构的混合计算。那么动态的连接方式就出现了。主要有多层总线连接，或者交叉开关。<br>
c宽带连接 （常见），这是我们能够用得起的，因为以上的连接方式有的需要专用设备，有的需要根据节点进行设计，而且随着以太网和交换机的发展，网络连接的速度可以满足通信需求，大部分的集群都采用宽带互联网。<br>

2.3存储
-------

随着cpu的发展，目前的瓶颈在于存储，所谓的内存墙（memory wall）解决方法就是增加缓存，增加缓存层级或者扩大缓存容量，在其中找到一个价格也能接受，速度也够用的平衡。<br>
![](https://pic3.zhimg.com/80/v2-951e08e8c059629161c680266790e442_1440w.webp)  <br>
而对于不同的需求和设计，也会有不同的存储管理机制， 如果是大规模并行系统(MPP)，数据超大，就需要单独的存储管理，对于小型集群，存储的设计可以单独设计NFS也可以直接利用节点存储。<br>

2.4超级计算机的分类
------

集群（cluster），利用商用节点，商用cpu或者gpu，采用商用交换机连接，操作系统采用linux，GNU编译器，和PBS。<br>

星群（constellation），系统的每个节点是一个并行机子系统，每个子系统里包含好多个处理器，采用商用交换机连接，操作系统和编译系统和PBS可以采用专用的。<br>

大规模并行系统（MPP，Massively Parallel Processing），每个节点包含10个左右的处理器，共享存储，处理器可以用专用的，或者商用的，采用高性能网络连接，节点分布存储。 操作系统和编译系统和PBS需要专用的。<br>

具体的超算信息可以查看top500 [TOP500 Lists | TOP500 Supercomputer Sites](https://www.top500.org/lists/top500/ "进入网站")<br>

------

3操作系统和软件环境
=====
