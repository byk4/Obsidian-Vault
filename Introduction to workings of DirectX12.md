For GPU programming one DX 11, DX 12, OpenGL and Vulkan APIs. But DX 11 and OGL are older APIs and they are deprecated. DX 12 and Vulkan are modern low level APIs that give you a lot of control over the GPU but are more difficult to apply. DX 11/ DX 12 only work on MS OS while OGL/Vulkan can be applied to Linux. 

For mobile we use metal, OpenGL ES and vulkan.  
- OpenGL ES and Vulkan work on android
- Metal is an API for Apple IOS, it's a low-level api and it also works on Mac
- There are ways to translate Vulkan into metal, so theoretically it is possible to program code in Vulkan and it will be carried out everywhere. 
But most of today's major video games use DX 12. The ideas are similar in Vulkan, but in Vulkan one has to write a lot of code to create a simple application. 

# DXGI - Driver Model

## DXGI Factory
DXGI is a library that allows DX 12 to communicate with Win 32 and the OS. User-mode drivers are the driver that performs most of the functions of DX 12 API. The kernel driver shader the GPU different applications that want to use it at the same time. 

## DX12 Device
Almost every computer has one or more graphics card. To access all of them we create an object like `DXGIFactory`. This object will allow us to get list of graphics adapters (GPUs) in our machine. The graphics adapter can be software adapter or hardware adapter. When we find the adapter we want to use (and which supports DX 12) we will create an object of `ID3D12Device` to initialize DirectX 12

## Debug Layer
We can also include a validation layer that will check if we are many any basic errors when using DX 12 API. This layers slows down the code, so use it only when in Debug Mode. This layer wont catch all our mistakes but it can be of great help to find out most of them. 

## Command Queue
The command queue is where we send out our command buffers to be executed. In [[What is GPU and how to program it]] we understood that GPU contains the command processor which has different kinds of command queue (graphics queue, copy queue) that are send to shader array. 
![[Pasted image 20250909202748.png]]

## DXGI - SwapChain
WE have to somehow link our device to the window to which we draw and to do this we need a swapchain. It's an object that stores pixel buffers for the window. We can choose color buffer formats (BGRA8, RGBA8, etc.) We can specify the number of buffers to store in swapchain. (*This is important if we want to use VSYNC in our display mode*) 
![[Pasted image 20250909203027.png]]

## Command allocator
We want to somehow select and fill command buffers and to do this we will create command allocator. 
- This is an object that allocates memory to commands
- The driver manages this memory for us.
This is our __COMMAND BUFFER__. After allocating the required memory we create a command list. Its is a linked to the command allocator, so deleting the allocator means that list can no longer be used

## Descriptors
To fill in the list of commands, GPU must know where our resources are in memory. This is present in `ID3D12Resource*`, but it exists in CPU. We must pass a pointer that will exist until the GPU finishes executing the list of commands, regardless what CPU does. For this DX 12 passes these pointers in form of descriptors

A descriptor is a pointer that stores everything it takes for the GPU to find an object and know it's metadata.
Ex. If clearing a buffer, the GPU must know its format. 

When we fill in the command list, and we need and fill a descriptor buffer, and correctly pass them to all commands that need them. In DX 12 there are 4 kinds of descriptors but for now, we will only use render target view handles that are used in the image cleanup command.  

![[Pasted image 20250909205249.png]]

## Command List - Cleaning
With a pixel buffer and handles for it, we can create our first command. To create a buffer/image clearing command call the `ClearRenderTargetView()` and pass handle of the buffer we want to clear. 

## Command List - Send 
The command list contains our commands and we are ready to send it to the GPU. First we need to close it using `commandList->Close()` Then we call `commandList->ExecuteCommandList()` which sends our lists to the GPU to execute. After that we call `SwapChain->Present()` which informs DXGI that in next frame, our new color buffer will be displayed on the monitor. 

## Fences
![[Pasted image 20250909205711.png]]

When working with command buffers in GPU programming, we often need a mechanism to know when commands have finished executing. This is important when CPU depends on GPU's progress or when we want to submit multiple command buffer in sequence and ensure the correct order. 

* A __FENCE__ is a synchronization primitive.  
- It acts as a __COUNTER__ or __STATUS FLAG__ that the CPU can query or wait on.
* The GPU will increment or update the fence value as it progresses through commands
* The CPU can check the fence value or wait until it reaches the desired value, indicating the commands are completed. 

Say we have two command buffer `buff1` and  `buff2`. We submit both to the GPU for execution. The CPU want to wait until __BOTH THE COMMAND BUFFERS ARE FINISHED EXECUTION__. 

Step 1: Initialize Fence: `fenceValue = 0`
Step 2: Submit Command Buffers with fence signal. 
- Submit `buff1` instructing GPU to increment fence when when `buff1` finishes
- Similarly, submit `buff2` instructing GPU to increment fence when when `buff2` finishes
Step 3: The CPU waits until the fence reaches the expected value, indicating both command buffers have completed.
Step 4: Proceed after Synchronization:
* Once the fence indicates completion, CPU proceeds safely, knowing that the command buffers are fully executed. 
## Command List - Closing
Once we know that the previous command list has finished execution (typically after waiting on a fence) we can safely reset the `CommandAllocator` by calling `ComamndAllocator->Reset()`. Then we call `CommandList->Reset(CommandAllocator, InitialPipelineState)` to prepare the command list for recording new GPU commands for the next frame. 

## Barrier
### Why do get errors about swapchain being in bad state?
When working with swap chain in DX 12, we often encounter errors like:
 
 >  "Resource in a bad state"
 
This happens because the resource state of buffers in GPU must be explicitly managed and transitioned between different stages of rendering pipeline.

### CPU vs GPU Resource layout
On the CPU when we create objects in memory (eg. Color buffers) the layout is always consistent: 
Pixels are stored in fixed position in memory, and we can read/write freely.
But on GPU the same resource can have different states depending it's usage. 
- State when writing (rendering to the buffer)
- State when reading (sampling from the buffer in shader)
- State when presenting (sending back to the display)
The GPU does not automatically know when a resource changes it's intended usage.

### What is a barrier?
A Barrier is a special command that:
1. Synchronizes different GPU commands.
2. Changes the resource's state explicitly. 
Without barriers the GPU could read and write to the same buffer simultaneously, leading to race conditions or undefined behavior.

### Swap Chain buffer example
At the start of the frame, the swapchain color buffer is in the __PRESENT__ state. Before we can clear it or render into it, we need to transition the resource to the __RENDER_TARGET state__ using barrier. 
After rendering, before calling `Present()` the resource must be transitioned back to the __PRESENT state__

### Synchronizing two drawing passes
We have 
1. __DRAW 0__: Renders into `Texture0`, it requires `Texture0` to be in __RENDER_TARGET__ state. 
2. __DRAW 1__: Reads from `Texture0` in the pixel shader, it requires `Texture0` to be in __PIXEL_SHADER_RESOURCE state__
To synchronize them properly:
* Insert a *resource barrier* between the two commands that transition `Texture0` from __RENDER_TARGET__ to __PIXEL_SHADER_RESOURCE__.
This ensures that all write from `Draw0` are fully committed before `Draw1` read the data.  

```c
#include <d3d12.h>
#include <dxgi1_3.h>

struct Rasterizer
{
	IDXGIAdapter1* adapter;
	ID3D12Device* device;
	ID3D12CommandQueue* commandQueue;
	IDXGISwapChain1* swapChain;
	U32 currentFrameIndex;
	ID3D12Resource* FrameBuffers[2];
	D3D12_CPU_DESCRIPTOR_HANDLE FrameBufferDescriptor[2];
	ID3D12CommandAllocator* commandAllocator;
	ID3D12CommandList* commandList;
	ID3D12Fence* fence;
	U64 fenceValue;
	ID3D12DescriptorHeap* RtvHeap;
};

Rasterizer createRasterizer(HWND windowHandle, U32 width, U32 height)
{
	Rasterizer result = {};

	IDXGIFactory2* factory = 0;
	ThrowIfFailed(CreateDXGIFactory2(IID_PPV_ARGS(&factory)));
	
	ID3D12Debug1* debug;
	ThrowIfFailed(D3D12GetDebugInterface(IID_PPV_ARGS(&debug)));
	debug->EnableDebugLayer();
	debug->SetEnableGPUBasedValidation(true);
	
	for(U32 adapterIndex = 0;
		factory->EnumAdapters1(adapterIndex, &result.adapter) != DXGI_ERROR_NOT_FOUND;
		++adapterIndex)
	{
		DXGI_ADAPTER_DESC1 desc;
		Result.Adapter->GetDesc1(&Desc);
		if(Desc.Flags & DXGI_ADAPTER_FLAG_SOFTWARE) continue;
		if(SUCCEEDED(D3D12CreateDevice(
			Result.adapter, D3D_FEATURE_LEVEL_12_0, 
			IID_PPV_ARGS(&result.device)
		))) break;
	}
	
	ID3D12Device* device = result.device;
	
	{
		D3D12_COMMAND_QUEUE_DESC desc = {};
		desc.Type = D3D12_COMMMAND_LIST_TYPE_DIRECT;
		desc.Flags = D3D12_COMMMAND_QUEUE_FLAG_NONE;
		ThrowIfFailed(device->CreateCommandQueue(&desc, IID_PPV_ARGS(&Result.commandQueue)));
	}
	
	return result;
}

```