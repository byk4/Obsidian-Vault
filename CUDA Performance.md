- __Maximum Utilization__: Maximize the number of warps that are running so we're executing as much as we can
- __Maximum Memory Throughput__: Optimize the memory usage so we're processing as much data as we can within a time frame
* __Maximize Instruction Throughput__: Optimize for instruction usage soo we're executing as many instructions as we can within a timeframe.

> Maximum Performance = Efficient data-parallel algorithms + Optimizations based on GPU architecture.

As a gpu programmer your job is not to build only parallel algorithms but to optimize it further to achieve the best performance possible

# Memory Coalescing
Threads in a warp are contiguous. They execute the same command at the same time. So why make them access same area of the memory. This is memory coalescing and works really fast. Basically concurrent/contiguous memory access is better.   
The concurrent accesses of the threads of a warp will coalesce into a number of transactions equal to the number of cache lines necessary to service all the threads. 
Loads and stores by threads of a warp can be combined into as low as one instruction