# Resource Monitoring IDs
Platform RDT artecture defined RIMDs
* Threads/Apps/VMs grouped into RDIMs for resource monitoring
* Any thread, app, VM or a combination can be monitored with any RMID
* Specify the RMID for a thread via the per-core [IA32_PQR_ASSOC](https://github.com/pengfwan0317/Intel-RDT/blob/master/brief_RDT/RDT%20MSR.png)("PQR") MSR
![image](https://github.com/pengfwan0317/Intel-RDT/blob/master/brief_RDT/RMID1.png)
#	RMID
OS或VMM会给每个应用或虚拟机标记一个软件定义的ID，叫做RMID（Resource Monitoring ID），通过RMID可以同时监控运行在多处理器上相互独立的线程，注意这里是指应用线程而是不是硬件的core，这个是由根本差异的。每个处理器可用的RMIDs数量是不一样的，这个可以通过CPUID指令获取得到，RMID数量不一样意味着可以监控独立线程的数量会有差异，如果一个系统上运行过多的线程可能会导致不能监控到所有线程的资源使用。
此外线程可以被独立监控，也可以按组的方式进行监控，多个线程可以标记为相同的RMID。同一个虚拟机下的所有线程可以标记为相同的RMID，同样一个应用下的所有线程可以标记为相同的RMID。绑定RMID到线程的动作由OS/VMM来完成。
#	RMID深入
每个core上存在一个MSR (IA32_PQR_ASSOC)，可以关联一个RMID（而一个RMID对应一个线程），该RMID就记录在这个MSR上，然后通过硬件监控资源使用率。其中CLOS用于资源分配后面再说。
因为RMID关联到了core，而应用线程关联到RMID。这样就开始监控线程的资源使用率了。那么监控的数据如何获取？
也是通过寄存器来实现，通过MSR (IA32_QM_EVTSEL) 选择寄存器中设置RMID和Event ID。
在软件设置了合理的RMID+Event ID后，硬件会查看指定的数据，并通过MSR (IA32_QM_CTR)返回。
其中E/U 位表示Error和Unavailable，当数据合法时不会设置这两个位。那么数据就可以被软件使用。
Intel官方文档中提示，后续RMID的含义会扩展，包含更多的资源监控。
#	RMID对OS的需求
这个里有个问题，就是如果线程发生调度到其他core，那么硬件core上的MSR (IA32_PQR_ASSOC)上所记录的RMID对应的线程并没有运行在本core上了，就会导致数据不准确了。所以希望OS/VMM支持，将RMID加入到应用线程状态结构体中，这样在线程切换的时候MSR (IA32_PQR_ASSOC)中RMID能自动更新，确保跟踪的正确性。
