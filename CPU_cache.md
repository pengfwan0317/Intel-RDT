![image](https://github.com/pengfwan0317/Intel-RDT/blob/master/Cache_in_RHEL.png)

|File         | explanation           | PO CPU0 index0 Value  |
| ------------- |:-------------:| -----:|
| coherency_line_size | size of each cache line usually representing the minimum amount of data that gets transferred from memory | 64 |
| level | represents the hierarchy in the multi-level cache      | 1 |
| number_of_sets | total number of sets, a set is a collection of cache lines sharing the same index  | 64 |
| physical_line_partition | number of physical cache lines sharing the same cachetag   | 1 |
| size | Total size of the cache     |  32K |
| type | type of the cache - data, inst or unified     | Data |
| ways_of_associativity | number of ways in which a particular memory block can be placed in the cache     |  8 |
| shared_cpu_list |       |    0,32 |
| shared_cpu_map |       |        |


00000000,00000000,00000000,00000000,00000000,00000000,00000000,00000000,00000000,00000000,00000000,00000000,00000000,00000000,00000001,00000001  
这个一个16进制表示，每个bit表示一个cpu，换算成2进制，1个数字可以表示4个cpu
ways_of_associativity*number_of_sets*coherency_line_size/1024 = size
