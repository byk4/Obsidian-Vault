#directx12
#gpu

GPU is a massive parallel processor that calculates trillions of operators per second and is used for graphics rendering. 

The GPU uses parallel execution to draw millions of triangles in real time. Much faster than serial CPU code. A CPU code may calculate 4 pixels at a time but a GPU can compute tens of thousands of pixels at a time

When we write programs for CPU, we allocate parts of memory from RAM. This is the memory that the CPU can read/write. Usually the GPU loads from it very slowly as we had seen in. Because of this GPU contains its own memory called VRAM/video memory. This memory is optimized for working in parallel. The CPU doesn't have easy access to it. All models/buffers that GPU is working on must be in VRAM in order to draw them quickly. 

But how can we program GPU to display graphics? 
First we start by allocating memory in VRAM for all our objects like Vertex Buffer, Depth buffer, etc. Then we copy the vertices and indices from the CPU to the GPU. There is a PCIe computer bus that transfers between CPU and GPU. After that we can draw our calls. In the CPU we build a command buffer that stores an array of graphics instructions. In each frame we build this buffer and when it's ready we send it to the GPU. The GPU will then process it in parallel manner with CPU working on instructions for the next frame. When GPU finishes the buffer it informs CPU and waits for the instructions to come from CPU. 

![[Pasted image 20250909173258.png]]

When we tell the GPU to draw, how does it really do it? There are two points of view:
- API point of view
- Processor (hardware) point of view

# API Point of View
## Input assembler
The first stage of graphics pipeline is called the input assembler. At this stage the GPU reads all of our vertices using the bound buffers and arguments in the draw command. The Draw command specifies the number of indices in the mesh, where to start reading indices and where to start reading vertices. 

![[Pasted image 20250909173630.png]]

## Vertex Shader
The vertices read are then send to the vertex shader stage. This is the first programmable stage in the graphics pipeline. Shader is a program the runs on the GPU just like CUDA kernels. Shaders are written  to process a single element but are called to process an array of elements. The input to shader is a single vertex and the output is a converted vertex. Ex. A simple multiplication of all position of all vertices with a matrix for transformation. 

```c
struct input_vertex
{
	float3 Position: POSITION;
	float2 Uv: TEXCOORD0; 
};

struct vertex_output
{
	float4 Position: SV_POSITION;
	float2 Uv: TEXCOORD0; 
};

float4x4 TransformationMatrix;

vertex_output VertexShader(input_vertex InputVertex)
{
	vertex_output Output;
	Output.Position = mul(TransformationMatrix, float4(InputVertex.Position, 1.0));
	Output.Uv = InputVertex.Uv;
	return Output;
}
```

## Post Processing Vertices
Now our vertices are transformed and send them to the next stage called Vertex Post-Processing. At this stage, we build triangles from vertices and cut off these triangles from the view frustrum and apply perspective (divide by w) and bring them to homogenous coordinates. The GPU handles everything for us

## Rasterizer
Now, our triangles are ready to be rasterized. GPU handles this for us and we don't have to write program for it. We can change the modes of rasterization but we don't have complete control over it. It's not programmable. This stage output fragments/pixels of triangles with interpolated attributes. 

## Pixel Shader
With these pixels we want to color them correctly and this is done by pixel shader stage. This is again programmable and we write a shader for it. It takes one pixel and returns the pixel color for it. Usually this stage, we calculate how light interacts with the material of the pixels. 

__HANDLING TEXTURES__ : The GPU contains built-in features that sample textures for us. These functions are optimized to run very quickly. The sampling function will automatically: 
- Convert texture formats into a format that shader will understand
- Execute various filter algorithms
- Texture border processing
We can use these functions in any shaders. 
```c
struct vertex_output
{
	float4 Position : SV_POSITION;
	float2 Uv : TEXCOORD0;
};

Texture2D<float4> DiffuseTexture; 
sampler BilinearSampler;

float4 PixelShader(vertex_output InputPixel): SV_TARGET
{
	float4 color = DiffuseTexture.Sample(BilinearSampler, InputPixel.Uv);
	return color;
}
```

## Output Merger
Finally we take our colored pixels and store them in a buffer in the output merger stage. This stage tests the depth first and if the pixel is visible then it stores it in the color buffer. The GPU Tests the buffer after rasterization, but only if the pixel shader does not change the pixel depth. Remember that this is a logical pipeline and the GPUs will execute it correctly but will also optimize it where they can.

One feature of this stage is the order to triangle is preserved. Lets say we have two triangles that have same orientation , position and depth buffer value but both of them overlap a little bit. What triangle should be drawn in the overlapping part?

Because of this the latest triangle will be drawn and it will override the greens pixels. In such a way GPU implements painters algorithm. 

```
Input assembler
Vertex Shader <- Programmable 
vertex post processing
Rasterizer
Pixel Shader <- Programmable
Output merger 
```

Before understanding the processors point of view lets first compare CPU and GPU
The CPU is built to quickly compute many long and complicated serial programs. As a result it large caches and branch prediction modules. Where as GPU is built to compute millions of small and simple programs with blazing speed. It expects user to have a lot of work that they can do in parallel. 

# Understanding components of GPU 
Before understanding processor's point of view we need to understand how GPU works and its components.
## SIMD
The first form of parallelism detected in GPUs is SIMD. SIMD in GPU usually has minimum 32 elements. In SIMD same instruction runs on every element, so writing one instruction on single element (one pixel, one vertex, ...) but running on batch of elements. Each SIMD element "executes" the entire programs and because of these we call these elements threads.

A single SIMD Unit executes 32 elements (threads) in parallel and has a command pipeline. A group of 32 parallel threads is called a warp. The SIMD unit maintains a queue of upto 16 warps ready to execute. When a warp encounters latency, condition branches or dependent instructions instead of stalling the hardware switches to execute another ready warp from the queue. Due to these less space is allocated on the die for complex branch prediction

Each warp needs a set of registers for storing its temporary data and execution state. The GPU provides a register file, a block of fast on-chip memory where register for all active warps are allocated. When a warp is scheduled, it is assigned a portion on the register file for it's private registers. If the register file if full - there isn't enough space to accommodate another warp then even if there's space in the warp queue the new wave won't be added. This condition is called register pressure. Register pressure limits the number of active warps thus reducing parallelism and preventing the GPU from fully hiding execution latencies.

## Texture Unit
The GPU contains texture sampling functions and they are called texture unit. The main tasks of this texture units is to: 
1. Convert texture formats that shaders will understand
2. Perform various filtering algorithms 
3. Handle texture borders
4. In modern processors, this units handle most of the instructions related to read/write to memory. 
The texture block uses a lot of memory so the GPUs give the L0 cache next to it feed it that memory
![[Pasted image 20250909184456.png]]

## Compute unit (CU)
To create CU the smallest block capable of computing work - we group many SIMD units together. Each SIMD executes 32 elements (threads) in parallel per cycle. Each maintains a queue of 16 warps. There a compute unit with say 2 SIMDs can:
- Execute 64 threads per cycle 
- Hold upto 32 warps in total in the queue
Since each warp has 32 threads, the compute unit can manage 32 x 32 = 1024 threads in parallel. 

## Shader Arrays/Engines/Clusters
The CU can handle a lot but it's not enough to fast enough for current games frame rate requirements. Because of this GPUs have large blocks called array of shaders. This blocks contains:
- 10 CU
- 2 Rasterizers that do the job in parallel
- 2 Primitive Units that perform the input assembler and post vertex processing stages. 
- 4 Renderer Backend/ROPs that perform output merging (write colors from pixels shader to color buffer)

## Putting it all together
And finally to build a GPU we connect a certain number of shader arrays and a Command processor
The command processor acts as a GPU controller 
- It reads the command buffer sent by the CPU
- It parses the commands into individual work items (eg.Draw calls)
- It send these work items to each shader array
Each shader array contains many CU that execute these work items in parallel. When the work is complete the command processor notifies the CPU with an end-of-task message (interrupt or status flag).
# Processor Point of View
![[Pasted image 20250909190158.png]]
1. The command processor loads the commands and tells GPU units which buffers to read and what to compute.
2. After loading the draw command, the command processor send the command  to "start drawing" to all array
![[Pasted image 20250909193451.png]]
3. "start drawing command" will invoke primitive blocks which will start loading all vertices from the vertex/index buffers. 
4. As soon as enough vertices are loaded to form primitive (triangle) it assembles them into primitives.
5. It then creates warps of work and send them to CU for parallel execution. At the same time in a fully pipelined manner the primitive block continuous to load the next set of vertices from memory to prepare the next batch of work. 
	- Warp dispatch and vertex loading happens in parallel, ensuring high throughput and efficient GPU utilization.
![[Pasted image 20250909193555.png]]
6. When the compute unit receives a warp it adds them to the warp queue of SIMD32s. These SIMD32s then compute the vertex shader for these warps so at this stage each warp has up to 32 vertices in it. 
7. When SIMD32 finishes it send the converted vertices back to the primitive block 
![[Pasted image 20250909193631.png]]
8. Primitive block receives results from different CUs and then performs vertex post processing
![[Pasted image 20250909193732.png]]
9. After that its send to rasterizer
10. If the pixel shader does not change the depth, the rasterizer test the depth using renderer backend
![[Pasted image 20250909193809.png]]
11. The rasterizer converts the triangles into pixels and packs them into warps which it send back to CUs
![[Pasted image 20250909193836.png]]
12. The CUs queue these warps again and perform compute pixel shader
13. Each warp in this stage stores upto 32 pixels itself.
14. When it finishes, the result (colored pixels) is send to renderer backend. 
![[Pasted image 20250909193903.png]]
15. The renderer backend looks at the order of triangles, do depth test and write the final result to the color/depth buffer
16. When they finish their job, they inform the instruction processor that they are idle.
17. The instruction processor accumulates these messages and notifies the CPU when all the command buffers are completed.

Related topic: [[CUDA Introduction]], [[SIMD]]