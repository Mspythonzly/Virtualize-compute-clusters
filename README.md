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

需要cpu，有缓存，有内存，还需要网络连接，专用的工业超算还需要专门设计节点，比如下图就是cray的节点。(图加载不出来的话不关我事)<br>
![](https://pic4.zhimg.com/80/v2-bcb98be32d7abc552534caf6bdd4c947_720w.webp)  <br>

[Performance Computer, XC Series Supercomputers](http://www.cray.com/products/computing/xc-series?tab=technology "进入网站")

2.2网络拓扑
----

有了节点，就需要对节点进行连接，评价一个网络的准则应该是，对于一定数量的节点，点对点的带宽越高，折半宽度越大，或者网络直径越小，点对点延迟越小，质量就越好。拓扑结构可以分为静态，动态，和宽带连接三类。<br>
a静态连接，就是对于固定数量的节点，单独设计的一套固定的物理连接，至于用什么线连接，都是对用户不可见的，具体取决于厂商或者设计者。连接的拓扑包括阵列（array），环（ring），网格（mesh），网格环（torus），树（tree），超立方体（hypercube），蝶网（butterfly），Benes网等等。详情可以参考其他文献，*《面向拓扑结构的并行算法设计与分析》李晓梅等。现在比较有名的超算一般采用超立方体（hypercube）四维立方体，长这样 (图加载不出来的话不关我事)<br>
![](https://pic4.zhimg.com/80/v2-61a0a029aa6139cdbc357aaebb270a23_720w.webp)
b动态连接，静态虽好，但是扩展比较困难，如果采用比较简单一维阵列，一个节点挂了，整台都挂了，而且不适合异构的混合计算。那么动态的连接方式就出现了。主要有多层总线连接，或者交叉开关。<br>
c宽带连接 （常见），这是我们能够用得起的，因为以上的连接方式有的需要专用设备，有的需要根据节点进行设计，而且随着以太网和交换机的发展，网络连接的速度可以满足通信需求，大部分的集群都采用宽带互联网。<br>

2.3存储
-------

随着cpu的发展，目前的瓶颈在于存储，所谓的内存墙（memory wall）解决方法就是增加缓存，增加缓存层级或者扩大缓存容量，在其中找到一个价格也能接受，速度也够用的平衡。 (图加载不出来的话不关我事)<br>
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

3.1操作系统
-----

都是UNIX和类UNIX，比如linux，操作都差不多，学好linux还是很重要滴。Windows也可以，但是目前已经不是主流。<br>

需要的语言，不分顺序，Fortran 77/90/95、c／c++、shell、python、perl、等等等等。因为在科学计算领域，软件更新比较慢，所以祖传老代码比较多，所以，使用标准一般比较老，且不保证兼容，所以想一次编译通过，不是很容易（头疼）。<br>

根据架构的不同，编译器一般会有选择，比如开源的GCC，Intel全家桶，ARM编译器，超算制造商的特殊编译器，IBM编译器，Cray的编译器，Fujitsu的编译器等等，开源的一般来说效率没有收费的高。<br>

3.2进程，进程通信
-----

进程（process）是基本的单位，包括，程序代码、控制状态、数据、执行状态，进程拥有独立的运行环境，高内聚低耦合嘛。在高负载的多核运算中，多个进程跑在一个物理核心上是不一定能缩短总运行时间的（walltime），有时候反而会小幅增加，所以，说核心的时候，尽量以物理核心为准。<br>

进程内的单位就是thread，如果利用openmp进行节点内并行的话，就需要创建thread。<br>

进程间的通信 有三种形式，通信、同步、和聚集，主要形式就是通信，通过消息（message）传递，进行通信，实现的形式可以是通过共享存储或者网络通信，但是这对用户都是不可见的，我们只需要调用MPI。<br>

4两台PC的“集群”
======



*两台节点：intel i7 4核，6g内存，1T硬盘，独立显卡（gpu太弱，不用它来计算，只用于连接显示器）intel Gigabit以太网卡。<br>

*网络连接：gigabit以太网switch hub（L2交换机），网线，需要路由器连接互联网<br>

4.2安装操作系统
------

安装ubuntu，具体的安装步骤就很简单，略。但是要注意的是尽量采用同样的用户名和密码，并将计算机名称编号，最后记得把用户设成管理员。<br>

4.3配置环境，安装软件
-----

```pseudocode
sudo apt -y update
sudo apt -y upgrade
```
<br>

首先，更新一下apt。
```pseudocode
sudo apt -y install emacs openssh-server openssh-client nfs-common libgomp1 openmpi-bin libopenmpi-dev openmpi-common update-manager-core
```
<br>
然后安装这些软件。。。

这些都是啥呢，我慢慢讲：

*emacs :编辑器(比较适合新手, 老手可以直接使用vi)<br>

*openssh-server openssh-client: openssh 的服务和客户端<br>

*nfs-common: 网络文件系统<br>

*libgomp1 :openmp 是一套支持跨平台共享内存方式的多线程并发的编程API<br>

*openmpi-bin libopenmpi-dev openmpi-common: openmpi，开源的mpi<br>

*update-manager-core: 更新用<br>

```pseudocode
sudo apt -y dist-upgrade
```
<br>
然后更新一下软件的依赖，防止出现依赖的遗漏。<br>

4.4网络配置
-----

a）网络连接，使用交换机集线器连接，连接方式多种多样，由于我们的组装的规模很小，所以，低于交换机的接口数的时候就把交换机作为行星网络的中心，如果需要多个交换机，就采用层状交换机结构。出于简单考虑，IP没有设成静态，直接采用路由器的hdcp功能，动态分配ip，如果要长期使用，请设置成静态ip。那么自动分配的ip分别是XXX.65.121.82和 XXX.65.121.102,分别对应的名称是server01和server02。更改静态ip需要更改 ／etc／network／interface 文件（Ubuntu18的这部分设置被更新了，需要采取其他方法）。<br>

b）查询ip地址，可以直接去gui的界面去看，也可以输入<ifconfig>命令，新的操作系统已经不默认安装ifconfig了， 可以试试<ip add>。<br>

c）更改／etc／hosts文件，把ip和名字对应上，这样操作起来比较方便，不用处处都输入ip。如果查看自己的hostname，可以查看／etc／hostname文件。<br>

d）然后重启network服务<br>
    ```pseudocode
    sudo /etc/init.d/networking restart
    ```
    <br>
    `测试两台机器是否能够ping通，如果可以，说明网络没问题，如果出现问题，检查网络连接和网络设置。`
    <br>
    e）用ssh生成密钥<br>
    ```pseudocode
    ssh-keygen
    ```
    <br>
    f）同步密钥，可以进行免密ssh登陆。<br>
    ```pseudocode
    cd ~/.ssh
cp id_rsa.pub authorized_keys
rsync -avz ~/.ssh XXX.65.121.102:~/.
    ```
    <br>
    再测试两台机器是否可以免密ssh登陆，如果不行，检查问题。<br>

这时候有人会问，我现在有两台机器，可以拷贝一下密钥，要是我有n台机器，岂不是相互都要拷贝n-1个密钥？解决的方法很简单就是大家都用一个密钥，这样进行访问的时候就不用互相交叉进行，而是由一个中心进行相互通信。当然这就失去了非对称加密的意义，理论上利用rsh的话会更好一些，加密毕竟会增加一些通信时间。<br>

`*因为mpi通信需要网络权限，最好关闭防火墙和网络管理器。`
<br>
  
4.5单机多核
=======
单机多核的计算有多种实现方法，包括手动创立进程，使用OpenMP或者MPI。<br>

还有一种叫 OpenSHMEM （PGAS的一种），这几种的区别就是，OpenMP是共享内存，通过内存联系，MPI是分布式内存，利用消息联络，而PGAS就是利用分布式内存然后让你用着像共享内存，他们之间是可以混合编程也可以单独编程，PGAS貌似依赖机器架构多一点，有一点点类似CUDA，不同厂家支持的还不一样，可以针对某一台超算进行优化。<br>

注意MPI的一个开源实现叫做OpenMPI，不要和OpenMP混淆。<br>
*下面首先利用MPI进行节点内并行。<br>
    进入terminal，输入
    ```pseudocode
    %emacs -nw test.c
    ```
    
   <br>
*还记得我们刚才举的c语言的mpi的helloworld例子吗，把它输入进去<br>

*然后`ctrl+x`，`ctrl+s` ，保存<br>

*然后`ctrl+x`，`ctrl+c` ，退出emacs<br>
    ```pseudocode
    mpicc test.c 
    ```
    <br>
    编译<br>
    
```pseudocode
%mpirun -np 2 a.out
Hello, World.  I am 0 of 2
Hello, World.  I am 1 of 2
```
  
	-------
	
    运行，-np就是使用几个节点进行计算，上面这是两个的结果
    
	--------
	
```pseudocode
%mpirun -np 4 a.out
Hello, World.  I am 0 of 4
Hello, World.  I am 1 of 4
Hello, World.  I am 2 of 4
Hello, World.  I am 3 of 4
%mpirun -np 8 a.out
Hello, World.  I am 0 of 8
Hello, World.  I am 1 of 8
Hello, World.  I am 3 of 8
Hello, World.  I am 4 of 8
Hello, World.  I am 5 of 8
Hello, World.  I am 6 of 8
Hello, World.  I am 7 of 8
Hello, World.  I am 2 of 8
```
    
    *利用OpenMP进行多核并行
    随着现在cpu的核数月来越多，充分的利用节点内的多核也是很重要的，OpenMP（Open Multi-Processing）是一套支持跨平台共享内存方式的多线程并发的编程API。

它的特点就是不需要特殊的代码结构，而是利用编译器来决定，而且共享内存的方式在某些算法中效率更高，至于openMP和MPI哪个效率更高，取决于平台和算法。

openmp.c的示范代码<br>
```c
    #include <stdio.h>
#include <omp.h>
int main (void)
{
  int x;
  int i;
  int a[8]={0,0,0,0,0,0,0,0};
  x = 2;
  #pragma omp parallel num_threads(8) 
  {
    if(omp_get_thread_num()==0)
      printf ("um_thds=%d\n", omp_get_num_threads());
    #pragma omp for
    for (i=0; i<omp_get_num_threads(); i++)
    {
      x++;
      a[i]++;
      printf ("um_thds=%d x=%d a=%d\n", omp_get_thread_num(), x, a[i]);
    }
  }
  printf ("x=%d\n", x);
  return 0;
}
```
                                   
	
编译的时候记得加上flag ，在gcc中是 -fopenmp<br>
	
```pseudocode
    gcc -fopenmp openmp.c
```

	
   openmp的并行数量由环境变量OMP_NUM_THREADS来控制比如export OMP_NUM_THREADS=value。注意上面这段代码上第19行x++是并行的，而默认openmp会将所有变量进行共享，所以如果不设成private，那么运行的结果就是不固定的。 <br>
	
```pseudocode
$ ./a.out 
um_thds=6 x=5 a=1
um_thds=2 x=9 a=1
um_thds=8
um_thds=0 x=10 a=1
um_thds=7 x=6 a=1
um_thds=4 x=7 a=1
um_thds=3 x=8 a=1
um_thds=5 x=4 a=1
um_thds=1 x=3 a=1
x=10
$ ./a.out 
um_thds=6 x=6 a=1
um_thds=4 x=4 a=1
um_thds=8
um_thds=0 x=7 a=1
um_thds=2 x=6 a=1
um_thds=1 x=3 a=1
um_thds=5 x=6 a=1
um_thds=3 x=6 a=1
um_thds=7 x=6 a=1
x=7
$ ./a.out 
um_thds=8
um_thds=0 x=9 a=1
um_thds=1 x=7 a=1
um_thds=3 x=3 a=1
um_thds=6 x=8 a=1
um_thds=7 x=7 a=1
um_thds=5 x=8 a=1
um_thds=4 x=8 a=1
um_thds=2 x=7 a=1
x=9
```
	
  
当然如果需要的时候可以混合编程，节点内用OpenMP，节点间用MPI，一般来说这样的效率比较高。至于GPU那又是另一回事。

4.6做好了，一个小型的集群！
-------
    
新建一个mpitest.c 文件
	
```c
#include <mpi.h>
#include <stdio.h>

int main(int argc, char** argv) {
  // Initialize the MPI environment
  MPI_Init(NULL, NULL);
  // Get the number of processes
  int world_size;
  MPI_Comm_size(MPI_COMM_WORLD, &world_size);
  // Get the rank of the process
  int world_rank;
  MPI_Comm_rank(MPI_COMM_WORLD, &world_rank);
  // Get the name of the processor
  char processor_name[MPI_MAX_PROCESSOR_NAME];
  int name_len;
  MPI_Get_processor_name(processor_name, &name_len);
  // Print off a hello world message
  printf("Hello world from processor %s, rank %d"
	 " out of %d processors\n",
	 processor_name, world_rank, world_size);
  // Finalize the MPI environment.
  MPI_Finalize();
}
```
	
   <br>
    拷贝编译完成的a.out，到另外一台机器，保证两台机器的a.out出现在同一个目录位置。
    
```pseudocode
    scp ~/a.out XXX.65.121.102:~/a.out 
```
    
    
然后编辑machinefile
	
```pseudocode
server01 cpu=4
server02 cpu=4
```
    

    然后在这个目录下运行a.out（确保a.out也在这个文件夹下）

	
```pseudocode
%mpirun --machinefile machinefile -np 8 a.out
Hello world from processor server01, rank 3 out of 8 processors
Hello world from processor server01, rank 0 out of 8 processors
Hello world from processor server01, rank 1 out of 8 processors
Hello world from processor server01, rank 2 out of 8 processors
Hello world from processor server02, rank 6 out of 8 processors
Hello world from processor server02, rank 5 out of 8 processors
Hello world from processor server02, rank 4 out of 8 processors
Hello world from processor server02, rank 7 out of 8 processors
```

    这样就是十分初级的集群的雏形，可以进行多机并列计算。
    

	






4.7NFS设置
-------
刚我们进行计算中的重要一步就是吧编译好的可执行程序copy到每一台机器上，并保证其目录的位置相同，为了避免这样的重复操作，我们最好使用共享的文件系统，最简单的方式就是设置NSF。依靠网络进行挂载可以保证每一台节点都可以访问相同的路径。<br>
先安装相关软件<br>
	
```pseudocode
sudo apt install nfs-utils nfs-utils-lib
```
<br>
设置nfs相关服务在操作系统启动时启动<br>
	
```pseudocode
sudo systemctl enable rpcbind
sudo systemctl enable nfs-server
sudo systemctl enable nfs-lock
sudo systemctl enable nfs-idmap 
```
<br>
	
启动nfs服务<br>
	
```pseudocode
sudo systemctl start rpcbind
sudo systemctl start nfs-server
sudo systemctl start nfs-lock
sudo systemctl start nfs-idmap 
```
服务器端设置NFS卷输出，即编辑 /etc/exports 添加：<br>
	
```pseudocode
sudo emacs /etc/exports
```
<br>
	
/nfs XXX.65.121.0/24(rw)<br>
*/nfs – 共享目录<br>
*<ip>/24 – 允许访问NFS的客户端IP地址段<br>
*rw – 允许对共享目录进行读写<br>
当然还有其他设置选项，比如insecure sync ...<br>
	
```pseudocode
sudo exportfs
```
<br>
这个是显示一下是否挂在成功<br>
	
```pseudocode
sudo service nfs status  -l
```
查看NFS状态
	
```pseudocode
sudo service nfs restart
```
重启NFS，这样，服务器端就设定结束了<br>

Linux挂载NFS的客户端非常简单的命令，先创建挂载目录，然后用 -t nfs 参数挂载就可以了<br>
```pseudocode
sudo mount -t nfs  xxx.168.0.100:/nfs /nfs
```
	
```pseudocode	
可以先查看<br>
```

```pseudocode	
showmount -e 192.168.0.100
```
	
如果要设置客户端启动时候就挂载NFS，可以配置 /etc/fstab 添加以下内容
	
```pseudocode
sudo emacs /etc/fstab
192.168.0.100:/nfs   /nfs defaults     0   0
```
	
然后在客户端简单使用以下命令就可以挂载
```pseudocode
sudo mount /nfs
```
