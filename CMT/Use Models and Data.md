# High-Level Monitoring Usage Models
Shared resources within a multiprocessor chip  may be managed through a combination of monitoring and allocation control hooks (Figure 1) in a closed-loop fashion to enable resource-aware application management. Resource monitoring provides increased visibility so that resource utilization can be tracked, application sensitivity to available resources can be profiled, and performance/resource inversion cases can be detected. 



Figure 1. Resource Monitoring is a critical component in any shared resource management system to enable informed resource allocation decisions in a dynamic datacenter environment.

The Intel's Cache Monitoring Technology feature provides Last-Level (L3) cache occupancy monitoring. A simplified view of the hardware/software interface model common to all usages is shown in Figure 2. Starting from the left of the diagram, the first step is to enumerate the presence of the features on the specific processor in question, which is accomplished via CPUID. Specific sub-leaves as discussed in a previous blog post [1] provide details on the level of CMT support and parameters which may change across processor generations including the number of Resource Monitoring IDs (RMIDs) supported.

An RMID allows threads, applications or VMs to be tracked using the CMT feature. As discussed in a previous post [1] each thread is associated with an RMID, and multiple threads may be associated with the same RMID. This means that all threads in an application, or all applications in a VM can be tracked with the same RMID, and the cache sensitivity of an entire application or VM can be determined (dynamically, in real-time, while running concurrently with other applications and/or VMs).

The association of a thread with an RMID is handled by the OS/VMM, which writes a per-thread register (IA32_PQR_ASSOC, or PQR for short) on swap-on to a thread (Step 2 in Figure 2). Note that details of software enabling and support are discussed in the next in the series.



Figure 2. The three-step process common across CMT use models - enumeration, association and reporting.

After a software-determined period of time, cache occupancy can be read back per-RMID via a pair of Model Specific Registers (MSRs), as described in detail in the second blog in the series [1].

The retrieved monitoring data is then scaled by software as described in the second blog [1] and can be used for a variety of purposes as discussed in subsequent sections.

# Specific Cache Monitoring Technology Usage Models
Across all usage models, CMT enables dynamic and simultaneous monitoring of many threads/apps/VMs through the use of RMIDs.

The primary set of use cases include:

Real-time profiling: Application performance vs. Cache Occupancy (Figure 3)
Detection of cache-starved applications (which can be migrated for better performance)
Advanced Cache-Aware Scheduling for better system throughput


Figure 3. Example Cache Monitoring Technology profiling deployment

Additional use cases possible with CMT include but are not limited to:

Long-term dynamic application profiling for aberration detection and software tuning/optimization
Precise and accurate cache sensitivity measurement without the need for simulators
Cache contention detection and measurement (including finding cache-starved applications or VMs amongst a large set of co-running applications/VMs)
Monitoring performance to SLAs
Optimal insertion of new applications on a cluster
Charging/bill-back
Administration: data can be aggregated and provided back to datacenter administrators to gauge the level of efficiency within the datacenter
A detailed example of a real-time profiling use case follows to illustrate the capabilities that CMT enables.

# Cache Monitoring Technology Example Data: Application Profiling
Through the use of CMT, applications can be monitored simultaneously while running on a platform. In the typical non-virtualized case shown in Figure 4 below, a number of applications are run on a 14-core Intel® Xeon®  E5-2600 v3 processor based system with RMIDs pinned to each core. As applications run, their cache occupancy can be sampled periodically. In Figure 4 periodic spikes in occupancy (green line) are visible from a periodic operating system task. In the middle of the plot a new memory streaming application is invoked on a core, which quickly consumes all of the L3 cache and then terminates. Using CMT this aggressor application can be detected, and if its behavior is found to interfere with more important applications, the aggressor application could be moved to another processor or another node. If the aggressor application is simply resource-hungry but high-priority then its true cache sensitivity can be measured over time using CMT (Figure 5, discussed later). 



Figure 4. CMT using one RMID pinned to each core, shown as a time series. The large spike in occupancy in the middle of the plot was caused by a memory-streaming application which quickly consumed all of the cache then exited. This data along with subsequent plots were generated from collected on an Intel® Xeon®  E5 v3 processor-based server.

Through the use of Cache Monitoring Technology the sensitivity of applications, especially cache-hungry applications can be measured dynamically, and a history can be constructed (Figure 5). 



Figure 5. Cache Sensitivity plotted using CMT for a variety of applications drawn from SPEC* CPU2006. 

Shown in Figure 5 is an aggregated set of data collected by sampling occupancy periodically for each application in the presence of a variety of other background applications running on a real system. The data was then aggregated by averaging all samples within a given range. In this case the ranges were selected by cache occupancy of 0-1MB, 1-2MB, 2-3MB, etc. (a simple “bucketing” scheme). The normalized application performance was then plotted vs. cache occupancy, providing direct visibility into the cache sensitivity of each application. As shown in the figure, povray, a ray-tracing application, is not cache sensitive (its performance is nearly constant across cache sizes). The bzip2 application is moderately cache-sensitive, showing sensitivity up to around 8MB of L3 cache (implying that applications should be scheduled to provide approximately this amount of cache to bzip2). The two remaining applications (mcf and bwaves) show significant cache sensitivity, and which implies that they should be given as much cache as possible to maximize their perfoprmance. There is some noise in the mcf data, caused by the highly variable and multi-phase behavior of mcf, however a good view of its cache sensitivity is clearly presented and if a more precise characterization is required, a longer sampling period could be used.

If an application’s cache sensitivity is curve-fit (typically using a logarithmic fit of the form y = coefficient * ln(x) + constant) and if the correlation coefficient is reasonably high then then the entire cache sensitivity of an application can be characterized and stored as a single pair of numbers, the coefficient and constant values.

More advanced analysis can also be conducted once the curve fit is in place. For instance, taking the derivative of the performance vs. occupancy (cache sensitivity) curve with respect to occupancy yields a curve providing cache sensitivity per unit of cache occupancy. By adding a threshold (Figure 6), cache sensitivity can be precisely derived for a given application, and a tunable optimal operating point can be determined. For instance, the optimal cache operating point might be defined as the point where adding an additional 1MB of L3 cache only increases application performance by 1%. Such thresholds can be used for dynamic scheduling, and for detecting cache-starved applications. 



Figure 6. Curve fitting application cache sensitivity (left side) then taking the derivative with respect to occupancy can generate a view of cache sensitivity vs. occupancy (right side), which can be used with simple thresholds to algorithmically measure the optimal cache operating point of any application. Here Instructions per Cycle (IPC) is used as a proxy for application performance.

The ability to dynamically measure the optimal cache operating point of an application is not possible without CMT, and is one of many new use models enabled with this technology. Without CMT the only way to obtain this data would be via simulation (not practical in a very dynamic datacenter environment) or estimation techniques (typically not practical due to inaccuracies and workload mutual interference in the datacenter).

The occupancy curves collected for various applications could be used to build long-term histories of applications and schedule optimally across sockets. For instance as shown on the left side of Figure 7 if two compute-intensive applications are co-located on a processor with small working sets (e.g., 4MB ideal cache size on a processor with 35MB L3 cache) then applications could be rebalanced across sockets to optimize L3 cache utilization and potentially increase performance (or stored for the next time the applications are run rather than dynamically moving them, simplifying the NUMA memory image implications). 



Figure 7. Rebalancing applications across processors for optimal cache utilization using CMT.

The example profiling data above shows only one use case of many.

Conclusion
Intel's Cache Monitoring Technology (CMT) feature enables threads, applications, VMs or any combination to be tracked simultaneously in a flexible manner to suit a wide variety of software usage models. The monitoring data collected can be used for application profiling, cache sensitivity measurement, cache contention detection, monitoring performance to SLAs, finding cache-starved applications, advanced cache-aware scheduling, optimal insertion of new applications, charging/bill-back and a variety of other advanced resource-aware scheduling optimizations. Data can be aggregated and provided back to datacenter administrators for instance to gauge the level of efficiency within the datacenter and guide future software optimizations.

Previous blogs are linked at the top of this page. The next blog discusses software enabling, tools and OS/VMM support availability. 
