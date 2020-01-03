1.	RDT前言
英特尔至强 E5-2600 v4在对外宣传时候号称“为云而生”的，除了其强大的性能和众多核心外，主要亮点就是Resource Director Technology（RDT）新技术的加入。使得其有理由宣称“为云而生”。
我们知道在一个虚拟化环境中，宿主机的资源（包括CPU cache和内存带宽）都是共享的。但是如果有一个消耗cache的应用快速消耗了L3缓存，或者一个应用消耗了系统大量内存带宽，那么如何保证其他虚拟机应用呢？如何限制这些“可恶”的邻居呢？
针对上诉问题，以前都是通过控制虚拟机逻辑资源来实现，但是调整的粒度实在太粗，针对处理器缓存这样敏感而稀缺的资源，几乎是无能为力的。为此英特尔推出了RDT技术，希望可以解决这个问题。
那么看下RDT到底是什么神奇技术。
2.	RDT技术组成
RDT技术有其实有5个功能模块，分别是
Cache Monitoring Technology (CMT)缓存监测技术、-- BDW
Cache Allocation Technology (CAT)缓存分配技术、-- BDW
Memory Bandwidth Monitoring (MBM)内存带宽监测技术、-- BDW
Memory Bandwidth Allocation (MBA)内存带宽分配技术、-- HSW
Code and Data Prioritization (CDP)代码和数据分区技术。 -- BDW
5个模块可以分为监控和控制两大类，CMT和MBM为监控技术，而CAT、MBA和CDP为控制技术。
RDT允许OS或VMM来监控线程，应用或VM使用的cache/内存带宽空间。通过分析cache/内存带宽使用率，OS或VMM可以优化调度策略提高效能，使得高级优化技术可以实现。
3.	为什么需要RDT
配合这几个技术，OS能够知道应用使用了多少CACHE空间，内存带宽，从而给虚拟机的虚拟处理器分配真实的CPU资源。结合CMT和CAT，缓存可做到实时监测和使用，能够让处理器的资源向虚拟机中最重要、最紧迫的任务分配。CDP可以限制数据在LLC中的存储，从而将空间节省出来给代码存储。
我们用更加直观的图来说明一下：
在一个系统里面运行多个应用，会出现应用线程之间LLC争用导致中断响应时间变长、性能吞吐量变小、高优先级进程资源被挤压等。
 
使用RDT后可以将LLC资源进行分配隔离，相互之间不会在出现争用。
 
目前的至强 E5-2600 V4（BDW平台）做到了对缓存的分配使用也即是CAT和CMT，并加入了对内存带宽的监测（MBM）。但是V4平台MBA技术是还没有实现的
Intel原计划是在下一代purley中实现，但是却在Purley 平台中出现对外宣称不支持,怪哉？。
4.	RDT具体功能
下面我们来看下RDT的一个具体功能。
以下方截图来说明，如下：
 
我们可以发现cores 0-5关联到了RMID 47-42,进 行了每个核监控。提供了CMT/MBM数据，从数据中发现core[
5.	RDT关键术语
下面介绍两个RDT引入的两个关键术语RMID和CLOS。
6.	RMID
OS或VMM会给每个应用或虚拟机标记一个软件定义的ID，叫做RMID（Resource Monitoring ID），通过RMID可以同时监控运行在多处理器上相互独立的线程，注意这里是指应用线程而是不是硬件的core，这个是由根本差异的。每个处理器可用的RMIDs数量是不一样的，这个可以通过CPUID指令获取得到，RMID数量不一样意味着可以监控独立线程的数量会有差异，如果一个系统上运行过多的线程可能会导致不能监控到所有线程的资源使用。
此外线程可以被独立监控，也可以按组的方式进行监控，多个线程可以标记为相同的RMID。同一个虚拟机下的所有线程可以标记为相同的RMID，同样一个应用下的所有线程可以标记为相同的RMID。绑定RMID到线程的动作由OS/VMM来完成。
7.	RMID深入
每个core上存在一个MSR (IA32_PQR_ASSOC)，可以关联一个RMID（而一个RMID对应一个线程），该RMID就记录在这个MSR上，然后通过硬件监控资源使用率。其中CLOS用于资源分配后面再说。
因为RMID关联到了core，而应用线程关联到RMID。这样就开始监控线程的资源使用率了。那么监控的数据如何获取？
也是通过寄存器来实现，通过MSR (IA32_QM_EVTSEL) 选择寄存器中设置RMID和Event ID。
在软件设置了合理的RMID+Event ID后，硬件会查看指定的数据，并通过MSR (IA32_QM_CTR)返回。
其中E/U 位表示Error和Unavailable，当数据合法时不会设置这两个位。那么数据就可以被软件使用。
Intel官方文档中提示，后续RMID的含义会扩展，包含更多的资源监控。
8.	RMID对OS的需求
这个里有个问题，就是如果线程发生调度到其他core，那么硬件core上的MSR (IA32_PQR_ASSOC)上所记录的RMID对应的线程并没有运行在本core上了，就会导致数据不准确了。所以希望OS/VMM支持，将RMID加入到应用线程状态结构体中，这样在线程切换的时候MSR (IA32_PQR_ASSOC)中RMID能自动更新，确保跟踪的正确性。
9.	CLOS
CAT中引入了一个中间结构叫做CLOS(Class of Service)，可以理解为资源控制标签。此外每个CLOS定义了CBM（capacity bitmasks），CLOS和CBM一起，确定有多少cache可以被这个CLOS使用。
一个应用可用的cache是通过一组MSR（IA32_L3_MASK_n,其中n表示CLOS数量）来指定的。
然后资源空间掩码（CBM）来标记相对可用空间、重复读和隔离情况。如下图中CLOS1 比CLOS3使用更少的cache,可以理解为更低的优先级。
对于LLC来说，Intel提供了CMT技术可以直接监控每个核心的LLC使用量，访问的命中、miss统计等关键性指标数据。而对应的CAT则可以对现有的LLC划分多个区块并在这些区块的基础上控制每个核心访问的区块从而实现了为不同的核心分配不同大小LLC的目的。这一部分的功能是相对完整且闭环的。
10.	RDT实战
前面讲述的是理论偏多，我们知道了RDT能干啥，接下去看下如何使用吧。
RDT使用分为两种方式，一种是直接将RMID绑定到硬件线程，然后将应用绑定到这些线程，第二种是使能OS/VMM调度（需要内核支持），在进程切换时候会自动将RMID进行更新，能够支持线程迁移。
我们使用Intel开源的工具来实现，不需要内核支持。通过这个软件包可以使用CAT,CMT,MBM,CDP功能。
工具软件下载链接如下：
https://github.com/01org/intel-cmt-cat
解压后，执行make && make install即可，如果找不到动态链接库，那么需要指定下动态库位置如下：export LD_LIBRARY_PATH=/usr/local/lib
RDT工具软件包的主要工具是pqos，pqos运行在用户层，通过标准Linux接口来访问MSR寄存器。因为msr文件接口是被保护的，所以需要root权限运行。支持在每个core或线程上提供CMT和MBM，其中MBM包括本地和异地内。
https://blog.csdn.net/weixin_30748995/article/details/98039811
https://blog.csdn.net/notbaron/article/details/75813942
https://software.intel.com/en-us/articles/introduction-to-the-intel-resource-director-technology-features-in-intel-xeon-processors-e5 
