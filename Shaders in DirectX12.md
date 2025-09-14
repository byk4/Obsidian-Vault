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

# Kinds of Descriptors
There are several kinds of descriptors used to pass data to the shaders

| Views                       | Descriptions                                             |
| --------------------------- | -------------------------------------------------------- |
| Constant Buffer View        | Describe the constant buffers<br>                        |
| Shader Resource View        | Describe textures and large constant buffers             |
| Unordered Access View (UAV) | Describe buffers/textures when shaders will value        |
| Sampler                     | Describe how to sample textures (eg. Bilinear filtering) |

# Passing Data to shaders

TO pass the input textures and buffers to the shaders we use root signatures. The root signatures describe how the descriptors are repackaged so shaders can access them in an organized manner. 

- It contains an array of *descriptor tables*
- Each descriptor table maps a range of descriptors from a heap to shader registers
- A root signature may contain many descriptors tables form different descriptor heaps

Additionally samplers can be passed in two ways:
1. Create descriptors that contains samplers
2. Define *static samplers* in root signature that are easier to implement 

In HLSL the `register()` keyword is used to assign a shader resource (line texture, sampler, constant buffer, UAV) to specific shader register slot.  

This tells compiler exactly where to bind that resource in the GPU pipeline allowing the application to know the binding location when creating descriptors in DX 12

`t0` means *Texture Register 0*. Bind Texture2D object named Texture to texture register slot 0

`s0` means *Sampler Register 0*. Bind `SamplerState` object named `BilinearSampler` to texture register slot 0


# Blending State
When drawing pixels we want the pixel shader result to overwrite the existing color buffer contents assuming the depth test passed and the pixel is closest to the camera.

Blending controls how the new pixel color and the existing color are combined

The resulting color is computed as:
>  ResultColor = BlendOp(SrcColor x SrcBlend, DestColor x DestBlend)
>  ResultAlpha = BlendOpAlpha(SrcAlpha x SrcBlendAlpha, DestAlpha x DestBlendAlpha)

For our purposes we will configure the blend state such that it just overwrites the new fragment color over the old one.  

