问题描述
运行大规模数据中心时，如何提高资源使用率是一个被广泛研究的话题，要提高资源使用率就绕不开资源隔离，每年都有很多论文和公司的实践经验发布。
对这个领域不了解的同学可能会问为什么这个问题这么热门，下面通过一个简单的计算来说明一下：
2013年9月xkcd估计谷歌有大约2百万台服务器（https://what-if.xkcd.com/63 ），按照数据中心即计算机（The Datacenter as a Computer）书中所说，每台服务器购买价只需要2000美元，加上其他运行成本，服务器资源使用率50%和使用率100%的综合成本相差约2600美元/台，按照2百万台服务器计算，哪怕是1%的使用率提高都是5200万美元。
现在你肯定会理解为什么各大公司对这个问题都如此热衷了吧。
提高资源使用率的最直接办法就是资源共享，资源共享最大的难题是如何有效隔离不同应用，使共享某种资源的应用互不干扰。
解决思路
在继续之前，我们需要先定义一下资源的概念，在计算机领域但凡是可以完成某种特定功能的组件都可以成为资源，大的方面有：
•	CPU
•	Cache，可细分为L1，L2，L3（Last Level Cache in Xeon platform）
•	Memory， 可细分为Bandwidth和Storage（可使用的内存大小）
•	Disk IOPS
•	Network
关于CPU、Memory Storage、Disk IOPS、Network的隔离现在有cgroups、namespace，虽然还不是很完美，但是一直在不断的改进中。谷歌的Borg系统算是最早使用cgroups的集群管理系统了，并贡献了cgroups代码给Linux。在日常的线上服务管理过程中，很少遇到这几方面隔离不好导致的问题，比较常见的是由于高吞吐量的batch任务thrash了缓存导致LS（Latency Sensitive）任务延迟升高。
缓存thrash
现在常见的CPU缓存一般有3层，第一层（L1）分指令缓存和数据缓存，第二层（L2）是指令和数据共享的，L1和L2位于同一个物理CPU上，如果该物理CPU支持超线程（HyperThreading），L1数据缓存是两个逻辑CPU共享的。第三层（L3），Intel Xeon系列里又称为LLC（Last Level Cache）, 同一个物理CPU package里所有CPU共享该缓存。
如下图所示的CPU缓存结构：
 
同一台服务器上的CPU数量在近几年一直在增加，32核->48核->72核，为了提高资源使用率，同一台机器上会同时运行很多应用，在同一个物理CPU的不同逻辑CPU上或者某个CPU package上不同物理CPU运行的应用会共享LLC，这样一个batch应用如果需要大量的读写LLC，会占据大部分LLC，导致LS应用在读取数据时产生cache miss。
 
L3 cache miss会导致160+ clock cycles的延迟，这对Latency Sensitive的应用来说是个灾难。
实现纯软件层面的缓存隔离，不仅复杂而且可能导致性能下降，原因是可能需要做内存的探测和page页的标记来实现LLC的分区，这方面的研究也有不少论文发表，但是真正在生产环境使用的暂时没有听说。
为了能够在不影响LS任务SLA的情况下，更好的共享计算资源，Intel引入了一种L3 cache的隔离技术：Cache Allocation Technology，简称CAT。
有理由相信，Intel一定是感受到了大型互联网公司在缓存隔离方面的痛苦（或者迫于压力，嘿嘿），才推出了该隔离技术。
CAT
该技术是在Intel的CPU微架构Haswell开始引入的，之后的CPU也都会支持。
 
CAT是RDT技术（Resource Director Technology）的一个子功能，目前 LLC是RDT支持的唯一资源。可以认为CAT使得LLC变成了一种支持QoS（Quality of Service）的资源，操作系统或者容器软件可以限制应用只能使用为其划分的部分cache区域，为不同应用划分的cache区域可以重叠。该功能在分配cache line时起作用，比如读取数据到cache的时候。
不同的cache区域是通过CLOS（Class of Service）标记来区分的，每个CLOS有一个CBM（Cache Bit Mask）。CBM是一个连续的bits，定义了每个cache区域可用的cache大小。
更多关于CAT相关的硬件实现细节，请查阅Intel Software Developer Manual的第17.16节。
使用场景
CAT可以在cloud和container环境下，更好的分配管理具有大LLC的高性能服务器。比如具有72个core的服务器上，需要同时运行web服务器，数据库和Map Reduce任务等。可以通过cache的分区隔离，使不同应用同时达到更好的资源共享效果。
因为CAT支持在应用运行时动态修改cache的分区，可以配合上层监控系统更好的优化LS的性能同时最小化对batch应用的影响。
一个典型的例子就是解决之前提到的noisy neighbor。在container环境下，一个运行在container里面的streaming应用不停的读写数据导致大量的LLC占用，会导致同机器上另外一个container里运行的LS需要的数据被evict出LLC，从而导致LS应用性能下降。
通过CAT，我们可以控制noisy neighbor可以使用的LLC大小，如下图所示：
 
如何使用
有了硬件上的支持，还需要软件层面的配合，kernel的支持（https://lkml.org/lkml/2015/10/2/72 ）是在2015年10月引入的。 在Linux kernel 4.6或者之后的版本，会有新的cgroup子系统‘intel_rdt’用来配置CAT。
查看CPU是否启用了CAT
cat /proc/cpuinfo # 检查flags是否包含’rdt’ 和 ‘cat_l3’
前面提到了LLC的区域划分是通过CLOS标记实现的，每个CLOS有一个CBM，CBM定义了可用的缓存区域。最大的CBM代表了整个LLC可用，通过‘intel_rdt’ cgroup子系统root node下的intel_rdt.l3cbm进行配置；child cgroup下的intel_rdt.l3_cmb默认继承parent cgroup的值，用户可用自定义覆盖。
可用通过以下步骤来启用CAT：
mount -t cgroup -ointel_rdt intel_rdt /sys/fs/cgroup/intel_rdt
mkdir cat_cgroup1
echo 0x0000f > /sys/fs/cgroup/intel_rdt/cat_cgroup1/intel_rdt.cache_mask #在一个CBM为20bits的系统上，这代表1/5的LLC
echo $PID > /sys/fs/cgroup/intel_rdt/cat_cgroup1/tasks
Linux下还支持把cpuset和intel_rdt两个子系统一起挂载，这样可用将CAT配合CPU亲和策略同时使用达到更好的性能。
结束语
CAT技术的应用可以在一定程度上解决CPU缓存thrashing的问题，可以保证LS应用在满足SLA的前提下，通过和batch应用共存达到提高资源使用率的目的。
但是如果贵司的生产环境使用的是原生的kernel，还需要耐心等待支持CAT的kernel发布之后才能够使用。
在共享资源的隔离上，Intel除了推出CAT技术，将来还会有memory bandwidth的隔离和power的隔离技术，让我们拭目以待吧！
https://software.intel.com/en-us/articles/introduction-to-cache-allocation-technology
