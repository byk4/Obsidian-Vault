Now that we have copied all data related to the model into GPU, we can create shaders and add commands to draw the model in every frame.

# Compiling Shader in DX 12

GPUs from different vendors and architectures cannot execute the same assembler code directly, because code formats vary between drivers. Therefore, the driver must compile shader code every time an application runs. However compiling HLSL directly is very slow and modern games often compile over __10000__ shaders!. To speed this up application first compiles HLSL to DXIL (DirectX Intermediate Language). DXIL is a bytecode format that is already somewhat optimized and easier for the driver to process. When the application provides DXIL the driver can convert it to GPU assembler code much faster than compiling HLSL directly. 

# Vertex shader example
In vertex shader we simply multiply each vertex position by a transformation matrix (World-View-Projection)

```c
struct vs_input
{
	float3 Position: POSITION;
	float2 Uv: TEXCOORD0;
};

struct ps_input
{
	float4 Position: SV_POSITION;
	float2 Uv: TEXCOORD0;
};

cbuffer RenderUniforms :register(0)
{
	float4x4 WVPTransform;
};

ps_input ModelVsMain(vs_input)
{
	ps_input Result;
	Result.position = mul(WVPTransform, float4(Input.Position, 1.0f));
	Result.Uv = Input.Uv;
	return Result;
}
```

This function takes vertex data and applies the transformation passing the result to the pixel shader.

# Pixel Shader Example

In the pixel shader, we sample the texture using bilinear interpolation and output the color to color buffer 

```c
Texture2D Texture : register(t0);
SamplerState BilinearSampler : register(s0);

struct ps_input
{
	float4 Position : SV_POSITION;
	float2 Uv : TEXCOORD0;
};

struct ps_output 
{
	float4 Color = SV_TARGET0;
}

ps_output ModelPSMain(ps_input Input)
{
	ps_output Result;
	Result.Color = Texture.Sample(BilinearSampler, Input.Uv);
	return Result;
}
```

# Using `ID3D12Pipeline`

Once shaders are compiled, we include them in our program using `ID3D12Pipeline` This object stores all data related to the graphics pipeline. When we issue a draw command in the command list, we bind the pipeline so the GPU knows how to process the model

There are several kinds of descriptors used to pass data to the shaders


| Views                       | Descriptions                                 |
| --------------------------- | -------------------------------------------- |
| Constant Buffer View        | Describe the constant buffers<br>            |
| Shader Resource View        | Describe textures and large constant buffers |
| Unordered Access View (UAV) | Describe buffers/textures when shaders will  |
