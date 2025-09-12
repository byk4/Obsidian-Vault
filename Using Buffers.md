# Memory Allocation

To create a depth buffer we need to allocate memory in the GPU. This can be done using `ID3D12Heap`. Heap is a segmented part of memory. In the CPU the heap is used for dynamic memory allocations. In GPUs there are different types of heaps that are used for different operations

For now we just want to allocate a depth buffer in VRAM, and we'll use the default heap for this. The default heap is the memory in the VRAM. The GPU can read/write to this memory. This is the fastest of all heaps. But CPU can't access it. 

First, use `CreateHeap()` to allocate memory for the depth buffer. The CPU tells the GPU that our application is using this part of the memory. Next call the `CreatePlacedResource()` to indicate that this memory contains a depth buffer. After that we can use this resource in the lists of commands that we need to send to the GPU. 
