We have created our first Dx 12 application that clears the color buffer. In this part we will create a depth buffer at the start and clear it every frame. After that we will see how to copy the sponza model to the GPU. 

# Allocation of memory to the depth buffer

To create a depth buffer we need to allocate memory on the GPU. This is done using an `ID3D12Heap`. A heap is a segmented part of the memory. ON the CPU heaps are generally used for dynamic memory allocation. On the GPU, there are different kinds of heaps used for different purposes. For now we need 