# Introduction
CMT is part of a larger series of technologies called Intel(r) Resource Director Technology (RDT). More information on the Intel RDT feature set can be found here, and an animation illustrating the key principles behind Intel RDT is posted here.

Key details discussed in this installment include Resource Monitoring IDs (RMIDs), an abstraction used to track threads/applications/VMs, the CPUID enumeration process and the Model-Specific Register (MSR) based interface used to retrieve monitoring data.

# Resource Monitoring IDs (RMIDs)
The CMT feature enables independent and simultaneous monitoring of many concurrently running threads on a multicore processor through the use of an abstraction known as a Resource Monitoring ID (RMID).

A per-thread architectural MSR (IA32_PQR_ASSOC at address 0xC8F) exists which allows each hardware thread to be associated with an RMID (specifying the RMID for the given hardware thread). 

![image](https://github.com/pengfwan0317/Intel-RDT/blob/master/CMT/IA32_PQR_ASSOC_MSR.png)

Figure 1. The per-thread IA32_PQR_ASSOC (PQR) MSR enables each thread to be associated with an RMID for resource monitoring. CLOS stands for Class of Service.  The CLOS field is used for control over resource allocation, which is beyond the scope of this article.

A plurality of independent RMIDs are provided, enabling multiple independent threads to be individually tracked. The number of available RMIDs per processor is one of the parameters enumerated in CPUID (see below).

Threads can be monitored individually or in groups, and multiple threads can be given the same RMID. This provides a flexible mapping (Figure 2) to span a wide variety of virtualized and non-virtualized usage models. 


![image](https://github.com/pengfwan0317/Intel-RDT/blob/master/CMT/RMIDs_logical.png)
Figure 2. Threads, applications, VMs or any combination can be associated with an RMID, enabling very flexible monitoring. As an example, all threads in a VM could be given the same RMID for simple per-VM monitoring.

Since each application or VM running on the platform consists of one or more threads, each application or VM can be monitored. For instance, all threads in a given VM could be assigned the same RMID. Similarly, all threads in an application could be assigned the same RMID. If the RMID is used only for monitoring that application (not a group of applications) then the occupancy reported by the system for that RMID will include only the specified application.

It is expected that in typical cases where an OS or VMM is enabled to support CMT, the RMID will simply be added to each thread’s state structure (Figure 3). Then when a thread is swapped onto a core, the PQR can be updated with the proper RMID to enable per-application or per-VM tracking.


![image](https://github.com/pengfwan0317/Intel-RDT/blob/master/CMT/thread_logical.png)
Figure 3. The PQR register (containing an RMID) stored as part of a thread or VCPU state, which is written onto the thread-specific registers when a software thread is scheduled on a hardware thread for execution.

Note that if a CMT-supported OS or VMM is not available, software may still make use of CMT by pinning RMIDs to cores, then carefully tracking which applications are allowed to run on which cores, which can be mapped to cache occupancy.

Additional details are available in [1].

The RMIDs described are a convenient resource tagging scheme which may be expanded in the future to encompass other resource types or functionality.

# Cache Monitoring Technology: CPUID Enumeration
The CPUID instruction is used to enumerate all CMT parameters which may change across processor generations, including the number of RMIDs available.

Typically an enabled OS or VMM would enumerate these capabilities and provide a standardized interface to determine the capabilities of these features by software running on the platform. This section gives a high-level overview of the details provided in CPUID.

The enumeration of CMT is hierarchical (Figure 4). To detect the presence of monitoring features in general on the platform, check bit 12 within CPUID.0x7.0 (a vector which contains bits to indicate the presence of multiple different types of features on the processor). 


![image](https://github.com/pengfwan0317/Intel-RDT/blob/master/CMT/CPUID_enumeration.png)
Figure 4. Hierarchical CPUID enumeration of monitoring features.

Once the presence of monitoring has been confirmed, the resources on which monitoring is supported can be enumerated through a new CPUID 0xF leaf. General information about which resources are supported is enumerated within CPUID.0xF.0 (note – subleaf zero is a special case which gives details about all monitoring features on the platform).

Once support for a particular resource has been confirmed, various subleaves (CPUID.0xF.[ResourceID]) can be polled to determine the attributes of each level of monitoring. For instance, L3 CMT details are enumerated in CPUID.0xF.1. Details enumerated include the number of RMIDs supported for L3 CMT, and an upscaling factor to use in converting sampled values retrieved from the Model-Specific Register (MSR) interface into cache occupancy in bytes.

Specific details about the leaves, sub-leaves and field encodings and details provided in CPUID are provided in [1].

# Cache Monitoring Technology: Model-Specific Register (MSR) Interface
Once support for CMT has been confirmed via CPUID and the number of RMIDs is known, each thread can be associated with an RMID via the PQR MSR RMID field (Figure 1).

After a period of time (as defined by the software) the occupancy data for a given RMID can be read back through a pair of keyhole MSRs which provide the ability to input an RMID and Event ID (EvtID) in a selection MSR, and the hardware retrieves and returns the occupancy in the data MSR.

 The event selection MSR (IA32_QM_EVTSEL) is shown in Figure 5. System software such as an OS or VMM retrieving monitoring data on behalf of an application or VM can program an RMID and Event ID pair corresponding to the type of data to be retrieved (for instance, L3 cache occupancy data for RMID). Available event codes are enumerated via CPUID and documented in [1]. 

![image](https://github.com/pengfwan0317/Intel-RDT/blob/master/CMT/IA32_QM_EVTESL_MSR.png)

Figure 5. The IA32_QM_EVTSEL MSR is used to select an RMID+EventID pair for which data should be retrieved. The data is then returned in the IA32_QM_CTR MSR (Figure 6).

Once the software has specified a valid RMID+Event ID pair, the hardware looks up the specified data, which is returned in the data MSR (IA32_QM_CTR, Figure 6). A pair of bits are provided in this MSR (Error and Unavailable) to indicate whether the data is valid or not. The precise meanings of these fields are documented in [1], however for the purposes of software if both bits are not set then the data in bits 61:0 is valid and can be consumed by the software. The error bits should always be checked before assuming that that data returned is valid. 


![image](https://github.com/pengfwan0317/Intel-RDT/blob/master/CMT/IA32_QM_CTR_MSR.png)
Figure 6. The IA32_QM_CTR MSR provides resource monitoring data for an RMID+EventID specified in the IA32_QM_EVTSEL MSR. If the E/U bits are not set then the data is valid.

In the case of the L3 CMT feature, the data returned from the IA32_QM_CTR MSR may be optionally multiplied by an upscaling factor from CPUID to convert to bytes before consumption in software. If software does not apply the upscaling factor the value returned is still useful for relative occupancy comparisons between applications/VMs however as the scale is linear.

Any monitoring features added in the future will make use of the same MSR interface, meaning that the software enabling effort is incremental, and would be limited to new CPUID leaves and monitoring event codes.

# Conclusion
Through the use of RMIDs, CMT feature enables threads, applications, VMs or any combination to be tracked simultaneously in a flexible manner to suit a wide variety of software usage models.

CPUID is used to enumerate all CMT parameters which may change across processor generations, including the number of RMIDs available, which is expected to increase over time.

One model-specific per-thread register is used to associate threads with RMIDs.  A pair of MSRs is used to retrieve monitoring resource data to enable the usage models described in the next blog in this series, which focuses on example data and use models.
