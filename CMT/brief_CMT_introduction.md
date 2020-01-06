# Benefits of Intel® Cache Monitoring Technology

## Introduction

The number of cores is increasing with the introduction of new processors.  As more cores are added, the number of diverse workloads that potentially can run simultaneously is also increasing.  Workloads can be single-threaded or multi-threaded applications and they can run in native or virtualized environments.  When threads, applications or virtual machines (VM) run concurrently, they place very different demands on shared resources like cache space and memory bandwidth. An example of this is when several VMs (each potentially running many applications) share cores on a processor, thus also sharing and contending for space in the last level (L3) cache.  This resource contention may result in inefficient space utilization, reduced performance and reduced performance determinism for applications on the system.  The Intel® Xeon® E5 2600 v3 product family introduces a feature in the hardware that allows monitoring of the last level cache (LLC) usage to help address these issues.

## What is Intel® Cache Monitoring Technology (CMT)?

CMT is part of a larger series of technologies called Intel(r) Resource Director Technology (RDT). More information on the Intel RDT feature set can be found here, and an animation illustrating the key principles behind Intel RDT is posted here.

CMT is a new feature that allows an Operating System (OS) or Hypervisor/virtual machine monitor (VMM) to determine the usage of cache by applications running on the platform. It currently monitors the L3 cache which is the LLC in most server platforms.

CMT provides mechanisms:

To detect if the platform supports this monitoring capability (via CPUID).

For an OS or VMM to indicate a software-defined ID for each of applications or VMs that are scheduled to run on a core. This ID is called the Resource Monitoring ID (RMID).

To monitor cache occupancy on a per RMID basis.

For an OS or VMM to read LLC occupancy for a given RMID at any time.

## How CMT Works

First, the platform and OS/VMM need to support this monitoring capability.  After this capability has been confirmed via CPUID, software should associate a thread or multiple threads or a VM with an RMID. Associating a thread with an RMID is done via a thread-specific model-specific register (MSR) which OS or VMM can program with an RMID value for a given thread.  After associating RMIDs with threads, software may execute for a period of time while hardware tracks occupancy before polling for the resulting occupancy.  Finally, event data for a given RMID (and thus a given thread, application, VM or a combination) can be read by system software at any time through an MSR interface consisting of two registers, one to specify the RMID and event code of interest, and the second to provide the data and an indication of whether any errors occurred or not.

## How CMT is used

There are several ways to take advantage of this feature to optimize the overall system performance.

Applications that are over-utilizing a particular level of cache in the hierarchy could be identified and migrated to another socket if needed.

Applications can be mixed and matched at the socket level to optimize the amount of shared cache available to each application at any given time, thus avoiding cache misses.

In a datacenter environment, cache utilization data can be used by system administrators to be aware of per-VM resource utilization and contention, and allows them to adjust scheduling policy decisions.

In some cases, performance histories can be built for applications to track effective cache space available vs. resulting application performance.  Later, if a specific performance target is required, cache-aware scheduling techniques can be used to ensure that an application has the necessary cache available to meet the performance target.

## CMT and Xen

There are online articles that describe how cache monitoring  is used with Xen.   Some steps are involved: first, one needs to detect if a system supports cache monitoring feature as well as how to initialize it .  The next step is to enable this feature in general and for each domain RMID.  It is important to collect cache monitoring Information from all Sockets. When configuring and running guest Operating Systems, the dynamically attach and detach cache monitoring service article will be very useful.

## Conclusion

CMT allows OS or VMMs to monitor how threads, applications or VMs use cache space.  By analyzing cache utilization, the OS or VMM can optimize scheduling policy decisions in order to improve overall system performance. Cache monitoring allows cache utilization to be simultaneously tracked for many concurrently running independent threads, applications or VMs at runtime, allowing advanced optimization techniques to be applied in real-time.
