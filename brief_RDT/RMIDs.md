# Resource Monitoring IDs
Platform RDT artecture defined RIMDs
* Threads/Apps/VMs grouped into RDIMs for resource monitoring
* Any thread, app, VM or a combination can be monitored with any RMID
* Specify the RMID for a thread via the per-core IA32_PQR_ASSOC("PQR") MSR
