#	RDT前言
我们知道在一个虚拟化环境中，宿主机的资源（包括CPU cache和内存带宽）都是共享的。但是如果有一个消耗cache的应用快速消耗了L3缓存，或者一个应用消耗了系统大量内存带宽，那么如何保证其他虚拟机应用呢？如何限制这些“可恶”的邻居呢？如何提升性能？
针对上诉问题，以前都是通过控制虚拟机逻辑资源来实现，但是调整的粒度实在太粗，针对处理器缓存这样敏感而稀缺的资源，几乎是无能为力的。为此英特尔推出了RDT技术，希望可以解决这个问题。
#	RDT技术组成
RDT技术有其实有5个功能模块，分别是 <br>
Cache Monitoring Technology (CMT)缓存监测技术、-- BDW <br>
[Cache Allocation Technology (CAT)缓存分配技术](https://github.com/pengfwan0317/Intel-RDT/blob/master/CAT/Cache%20Allocation%20Technology.md)、-- BDW <br>
Memory Bandwidth Monitoring (MBM)内存带宽监测技术、-- BDW <br>
Memory Bandwidth Allocation (MBA)内存带宽分配技术、-- HSW <br>
Code and Data Prioritization (CDP)代码和数据分区技术。 -- BDW <br>
5个模块可以分为监控和控制两大类，CMT和MBM为监控技术，而CAT、MBA和CDP为控制技术。
RDT允许OS或VMM来监控线程，应用或VM使用的cache/内存带宽空间。通过分析cache/内存带宽使用率，OS或VMM可以优化调度策略提高效能，使得高级优化技术可以实现。
#	为什么需要RDT
配合这几个技术，OS能够知道应用使用了多少CACHE空间，内存带宽，从而给虚拟机的虚拟处理器分配真实的CPU资源。结合CMT和CAT，缓存可做到实时监测和使用，能够让处理器的资源向虚拟机中最重要、最紧迫的任务分配。CDP可以限制数据在LLC中的存储，从而将空间节省出来给代码存储。
在一个系统里面运行多个应用，会出现应用线程之间LLC争用导致中断响应时间变长、性能吞吐量变小、高优先级进程资源被挤压等。
 使用RDT后可以将LLC资源进行分配隔离，相互之间不会在出现争用。 
目前的至强 E5-2600 V4（BDW平台）做到了对缓存的分配使用也即是CAT和CMT，并加入了对内存带宽的监测（MBM）。但是V4平台MBA技术是还没有实现的
Intel原计划是在下一代purley中实现，但是却在Purley 平台中出现对外宣称不支持,怪哉？。



#	RDT实战
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
