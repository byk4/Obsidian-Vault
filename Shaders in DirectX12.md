Now that we have copied all data related to the model into GPU, we can create shaders and add commands to draw the model in every frame.

# Compiling Shader in DX 12

GPUs from different vendors and architectures cannot execute the same assembler code directly, because code formats vary between drivers. Therefore, the driver must compile shader code every time an application runs. However compiling HLSL directly is very slow and modern games often compile over __10000__ shaders!. To speed this up application first compiles HLSL to DXIL (DirectX Intermediate Language). DXIL is a bytecode format that is already somewhat optimized and easier for the driver to process. When the application provides DXIL the driver can convert it to GPU assembler code much faster than compiling HLSL directly. 

