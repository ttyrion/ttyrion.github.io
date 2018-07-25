---
layout:     page
title:      【Direct3D 11】Build Application with D3D 11(with VS 2015) to support XP
subtitle:   让D3D 11应用程序也能在XP上跑起来（并非让XP支持D3D 11）
date:       2018-07-16
author:     翼
header-img: image/bg.jpg
catalog: true
tags:
---

# 前言
>  请注意，我这里是要开发一个D3D 11 应用程序，并且让该程序也能在XP上跑起来（基于Windows SDk 7.1A），并不是说让XP支持D3D 11。

#我的目的
1. 使用Direct3D 11渲染。
1. 我的应用程序要能支持XP。
1. 不支持Direct3D 11 的系统，可以使用D3D 9 或者 GDI。
1. 我不想装一堆SDK，既然已经按照了Visual Studio 2015（对于2012、2013是一样的），就不想再安装独立版DirectX 11 SDK。

说明：这里主要就是说一下，怎么用 VS 2015 开发一个D3D 11应用程序，还能在XP上启动。

#关于DirectX SDK的那些破事儿
这里主要提一下，Windows SDK 8.0（不支持XP）以后，DirectX SDK 已经不再独立发布，会随着Windows SDK 一起发布。详细情况，请参考我另一篇《关于DirectX 11 SDK的变动》。

了解了DirectX SDK的前前后后之后，总结一下有如下影响我们的信息：
1. Windows SDk 8.0 包含了DirectX 11 头文件和库等,但是它不支持XP，因此我们不能基于Windows SDk 8.0。
1. Visual Studio 2015（以及2012、2013）安装时会带着 Windows SDK 7.1A （支持XP）：它包含之前随着Windows SDK 7.1发布的头文件、库、以及一套工具子集。并且，它包含着比 Windows 8.x SDK 更旧的一套Direct3D 10、Direct3D 11 头文件。
1. Visual Studio 2015 项目的“平台工具集”配置中选择“v140_xp”（对于VS2012,选择“v110_xp”），该项目就会基于 Windows SDK 7.1A，才能支持 XP。
1. MSDN并不建议我们使用“v140_xp”这个配置开发。

无奈，公司的项目不得不使用“v140_xp”以便继续支持XP，并且尝试使用Direct3D 11 渲染视频（属于小项目，顺便给之后的大项目积累经验）。

#基于 Windows SDK 7.1A 开发 Direct3D 11 遇到的问题
1. **找不到DirectXMath.h**  
Windows 8.x SDK 里面的 DirectXMath 是兼容XP的，但是, "v140_xp" 平台工具集的包含路径中并没有 DirectXMath。但是微软已经将DirectXMath开源道[这里，GitHub](https://github.com/Microsoft/DirectXMath)上，所以我们可以直接到GitHub上Clone一份到本地，把头文件复制到我们的项目中。
2. **找不到d3dcompiler.h，没有D3DCompileFromFile等API可以调用**  
既然不能调用API编译着色器，我们能想到的当然是不编译就好了。确实，我们可以不用API编译，让开发工具替我们编译好。编译HLSL的工具是fxc.exe,但是无奈，"v140_xp" 平台工具集的包含路径中依然找不到它。
可以把Windows SDK 8.0的可执行文件路径（比如"C:\Program Files (x86)\Windows Kits\8.0\bin\x86"）总的fxc.exe 和 d3dcompiler_46.dll拷贝到项目某个子目录如tools中，并把tools目录设置到项目的可执行文件包含路径中。
3. **着色器代码编译失败**  
如果要改为使用fxc.exe编译着色器，着色器代码文件得做一些改动了。之前使用D3DCompileFromFile在运行时编译着色器时：
   ```cpp
   ID3DBlob* vertex_shader_blob = NULL;
   ID3DBlob* pixcel_shader_blob = NULL;
   ID3DBlob* err_blob = NULL;
   HRESULT hr = D3DCompileFromFile(L"../DXViewer/I420Frame.hlsl", NULL, NULL, "I420FrameVertexShader", "vs_5_0", D3DCOMPILE_DEBUG, 0, &vertex_shader_blob, &err_blob);
   if (FAILED(hr)) {
        char* msg = err_blob == NULL ? NULL : (char*)err_blob->GetBufferPointer();
        FailedDirect3DDebugString(hr, false, L"compile vertex shader failed.");
    }
   hr = D3DCompileFromFile(L"../DXViewer/I420Frame.hlsl", NULL, NULL, "I420FramePixelShader", "ps_5_0", D3DCOMPILE_DEBUG, 0, &pixcel_shader_blob, &err_blob);
   if (FAILED(hr)) {
     char* msg = err_blob == NULL ? NULL : (char*)err_blob->GetBufferPointer();
     FailedDirect3DDebugString(hr, false, L"compile pixel shader failed.");
   }
   ```
   I420Frame.hlsl代码内容如下：
   ```cpp
   cbuffer MatrixBuffer
   {
      //matrix world;
      //matrix view;
      //matrix projection;
      
      matrix mvp;
   };

   struct VertexShader_INPUT
   {
      float3 position : POSITION;
      float4 color : COLOR;
      float2 tex      : TEXCOORD0;
   };

   struct PixelShader_INPUT
   {
      float4 position : SV_POSITION;
      float4 color    : COLOR;
      float2 tex    : TEXCOORD0;
   };

   PixelShader_INPUT I420FrameVertexShader(VertexShader_INPUT input)
   {
      PixelShader_INPUT output;
      output.position.w = 1;
      output.position.x = input.position.x;
      output.position.y = input.position.y;
      output.position.z = input.position.z;

      //output.position = mul(output.position, world);
      //output.position = mul(output.position, view);
      //output.position = mul(output.position, projection);
      
      output.position = mul(output.position, mvp);

      output.color = input.color;
      output.tex = input.tex;

      return output;
   }

   SamplerState sample_state;
   Texture2D tex_y;
   Texture2D tex_u;
   Texture2D tex_v;
   float4 I420FramePixelShader(PixelShader_INPUT input) : SV_TARGET
   {
      float y = tex_y.Sample(sample_state, input.tex).r;
      float u = tex_u.Sample(sample_state, input.tex).r - 0.5f;
      float v = tex_v.Sample(sample_state, input.tex).r - 0.5f;
      float r = y + 1.14f * v;
      float g = y - 0.394f * u - 0.581f * v;
      float b = y + 2.03f * u;

      //return float4(input.color.r, input.color.g, input.color.b, 1.0f);
      
      return float4(r, g, b, 0.0f);
   }
   ```
   可见，我们是可以把顶点着色器和像素着色器放在一个hlsl文件中的。
   在使用fxc编译时，就要一个着色器一个hlsl文件了：因为VS中hlsl文件的着色器类型配置项，只能选择某一种着色器类型。
   我这里就把上面的着色器代码分成两个hlsl文件：
   **顶点着色器I420FrameVertex.hlsl：**
   ```cpp
     cbuffer MatrixBuffer
     {
         //matrix world;
         //matrix view;
         //matrix projection;
         
         matrix mvp;
     };

     struct VertexShader_INPUT
     {
         float3 position : POSITION;
         float4 color : COLOR;
         float2 tex      : TEXCOORD0;
     };

     struct PixelShader_INPUT
     {
         float4 position : SV_POSITION;
         float4 color    : COLOR;
         float2 tex    : TEXCOORD0;
     };

     PixelShader_INPUT I420FrameVertexShader(VertexShader_INPUT input)
     {
         PixelShader_INPUT output;
         output.position.w = 1;
         output.position.x = input.position.x;
         output.position.y = input.position.y;
         output.position.z = input.position.z;

         //output.position = mul(output.position, world);
         //output.position = mul(output.position, view);
         //output.position = mul(output.position, projection);
         
         output.position = mul(output.position, mvp);

         output.color = input.color;
         output.tex = input.tex;

         return output;
     }
   ```
   **像素着色器I420FramePixel.hlsl：**
   ```cpp
    struct VertexShader_INPUT
    {
        float3 position : POSITION;
        float4 color : COLOR;
        float2 tex      : TEXCOORD0;
    };

    struct PixelShader_INPUT
    {
        float4 position : SV_POSITION;
        float4 color    : COLOR;
        float2 tex    : TEXCOORD0;
    };

    SamplerState sample_state;
    Texture2D tex_y;
    Texture2D tex_u;
    Texture2D tex_v;
    float4 I420FramePixelShader(PixelShader_INPUT input) : SV_TARGET
    {
        float y = tex_y.Sample(sample_state, input.tex).r;
        float u = tex_u.Sample(sample_state, input.tex).r - 0.5f;
        float v = tex_v.Sample(sample_state, input.tex).r - 0.5f;
        float r = y + 1.14f * v;
        float g = y - 0.394f * u - 0.581f * v;
        float b = y + 2.03f * u;

        //return float4(input.color.r, input.color.g, input.color.b, 1.0f);
        
        return float4(r, g, b, 0.0f);
    }
   ```
   **上面的代码，其实是没有任何改动，仅仅只是把一个文件分成两个文件。**
   这两个hlsl文件编译会在输出目录生成.cso文件：I420FramePixel.cso和I420FrameVertex.cso。这就是二进制的着色器代码，我们可以直接把它们传给CreateVertexShader和CreatePixelShader：
   ```cpp
    std::string pixel_shader_binary = LoadShader("I420FramePixel.cso");
    std::string vertex_shader_binary = LoadShader("I420FrameVertex.cso");

    ID3D11VertexShader* vertex_shader = NULL;
    HRESULT hr = engine_->GetDevice()->CreateVertexShader(vertex_shader_binary.data(), vertex_shader_binary.size(), NULL, &vertex_shader);
    FailedDirect3DDebugString(hr, false, L"create vertex shader failed.");
    ID3D11PixelShader* pixel_shader = NULL;
    hr = engine_->GetDevice()->CreatePixelShader(pixel_shader_binary.data(), pixel_shader_binary.size(), NULL, &pixel_shader);
    FailedDirect3DDebugString(hr, false, L"create pixel shader failed.");
    engine_->GetDeviceContext()->VSSetShader(vertex_shader, NULL, 0);
    engine_->GetDeviceContext()->PSSetShader(pixel_shader, NULL, 0);
   ```
   上面的LoadShader，其实就是读取文件：
   ```cpp
    std::string LoadShader(const std::string& cso) {
        std::string shader;
        std::ifstream ifs;
        ifs.open(cso, std::ios::binary | std::ios::in);
        if (ifs.is_open()) {
            ifs.seekg(0, std::ios_base::end);
            int size = (int)ifs.tellg();
            ifs.seekg(0, std::ios_base::beg);

            shader.resize(size);
            ifs.read(&shader[0], size);
            ifs.close();
        }

        return shader;
    }
   ```
   着色器的问题已经解决。

至此，基于Visual Studio 2015自带的SDK开发支持XP的D3D 11应用程序的问题已经解决。剩下的是当判断出系统不支持DirectX 11（D3D_FEATURE_LEVEL_11_0）时的处理，这个就不再是疑难杂症了。
