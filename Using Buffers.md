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

Processors perform better when pointers are aligned to the power of 2. In GPUs alignment isn't just for performance; it's often mandatory. Some resources will only work if their starting address is properly aligned

For ex. If the alignment is 8 bytes that start address plus `Used` must always be multiple of 8.

We can use bit operations to calculate the aligned address. The general rule is:

``` c
nextAlignedAddress = (currentAddress + alignment - 1) & ~(alignment-1)
```

This ensures proper alignment regardless of alignment size. 

# Descriptor Management

We have used descriptor heaps for color buffers. Now, we will create a similar structure for depth buffer descriptors to manage them efficiently, similar to linear arena but storing an array of descriptors

# Copying Model Data to the GPU
We want to load the Sponza model resouces (textures, vertex buffers, index buffers, etc.) into the GPU. 

- The Upload heap is special RAM where the CPU can read/write and the GPU can read. However GPU can access from the Upload heap is slower than access to the default heap. 

- We use upload heap to stage data and then instruct the GPU to copy it to resources in default heap (VRAM) 

So the full data copy process looks like
1. CPU -> Upload Heap (RAM)
2. Upload Heap -> Default Heap (VRAM)

Copy from GPU to CPU is similar but it uses *Read Back Heap*

Since this process is slow it's important to minimize data transfers between CPU and GPU unless absolutely required. 

## STEPS TO IMPLEMENT DATA COPYING

1. Create all model resources using __LINEAR AREANA__ in the __DEFAULT HEAP__
2. Independently allocate space in the __UPLOAD HEAP__ for each resource.
3. For each resource, store the data to be copied in the __UPLOAD HEAP__ 
4. Record the GPU copy commands in the __COMMAND LIST__ 
5. Before rendering the frame, execute the __COMMAND LIST__ and wait for the copy to finish
6. Clear the __UPLOAD ARENA__ (reset `Used` to zero) to reuse the memory for future transfers  

### Special Case: Copying textures

It's slightly more complicated that buffers. Each texel row occupies a certain amount of memory. DX 12 requires the number of bytes per texel row be a multiple of `D3D12_TEXTURE_DATA_PITCH_ALIGNMENT` (256 bytes) when in the __UPLOAD HEAP__

For ex. If row is not multiple of 256 bytes padding must be added to meet the requirements.
