We have created our first Dx 12 application that clears the color buffer. In this part we will create a depth buffer at the start and clear it every frame. After that we will see how to copy the sponza model to the GPU. 

# Allocation of memory to the depth buffer

To create a depth buffer we need to allocate memory on the GPU. This is done using an `ID3D12Heap`. A heap is a segmented part of the memory. ON the CPU heaps are generally used for dynamic memory allocation. On the GPU, there are different kinds of heaps used for different purposes. For now we want to allocate a depth buffer in VRAM so we will use *default heap*

The *default heap* refers to memory located in VRAM. Compared to the other heaps it allows the GPU to read and write the fastest. However CPU cannot directly access this memory. 
1. We need to use `CreateHeap()` to allocate memory for the depth buffer. This informs the GPU that our application will use this section of memory. 
2. Call `CreatePlacedResource()` to specify that this memory contains a depth buffer. 
3. Use this resource in our command list to sent to the GPU. 

Although we can allocate memory separately for each resource this is a bad practice because the resources would be scattered in memory leading to more cache misses.  

# Using Linear Allocation for Efficient Memory Access

Instead of allocating memory separately for each resource, we can allocate once and place many resource within that single allocation. To achieve this we use *Linear Arena*

A *Linear Arena* holds a large memory block provided by the OS or DX 12. When the application needs memory for a resource, it requests a portion of the arena, which returns an allocation section. If there's no space left, an error is triggered.  

In a typical application many blocks could be added, but for simplicity we'll create a large enough block to hold all our resources. 

One important characteristic of a linear arena is that it cannot easily release individual allocations. The arena doesn't track each allocations details. Instead, we simply clear the entire arena by resetting the `Used` size to 0. Future allocations will overwrite the previous ones.

# Memory Alignment

