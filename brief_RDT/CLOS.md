# Key Concept: Classes of Service(CLOS)
Platform RDT atchitecture defined RMIDs and CLOS
* Threads/Apps/VMs grouped into Classe of Service(CLOS) for resource monitoring
* Resource usage of any app, VM or a combination can be monitored with a CLOS.
* Specify the CLOS for a thread via the per-core IA32_PQR_ASSOC("PQR") MSR
![image](https://github.com/pengfwan0317/Intel-RDT/blob/master/brief_RDT/CLOS.png)
#	CLOS
CAT中引入了一个中间结构叫做CLOS(Class of Service)，可以理解为资源控制标签。此外每个CLOS定义了CBM（capacity bitmasks），CLOS和CBM一起，确定有多少cache可以被这个CLOS使用。
一个应用可用的cache是通过一组MSR（IA32_L3_MASK_n,其中n表示CLOS数量）来指定的。
然后资源空间掩码（CBM）来标记相对可用空间、重复读和隔离情况。如下图中CLOS1 比CLOS3使用更少的cache,可以理解为更低的优先级。
对于LLC来说，Intel提供了CMT技术可以直接监控每个核心的LLC使用量，访问的命中、miss统计等关键性指标数据。而对应的CAT则可以对现有的LLC划分多个区块并在这些区块的基础上控制每个核心访问的区块从而实现了为不同的核心分配不同大小LLC的目的。这一部分的功能是相对完整且闭环的。
