
---
title: "UE调试Shader编译失败的问题"
date: 2024-03-30T00:20:49+08:00
draft: false
categories: [ "UE"]
isCJKLanguage: true
slug: "04dbd0cf"
toc: true
mermaid: false
fancybox: false
blueprint: false
# latex support
# katex: true
# markup: mmark
# mmarktoc: false 
---


{{% notice info %}}
Engine Version: 5.2.1
{{% /notice %}}

最近碰到了打包UE的时候碰见了编译Spir-v Shader的时候，DXC直接抛Internal Error然后导致ShaderCompileWorker进程崩溃的情况，这种情况下甚至不会导出编译失败的Shader的源码。
花了一段时间排查解决了，记录一下备忘。

有一些自定义编译Shader编译过程参数的时候也可以参考下面的备忘。

调试Shader编译错误的时候，我们需要

- 正在编译的Shader源码
- 当前的编译选项
- 编译结果和错误警告信息

# ConsoleVariables.ini 禁用单独进程编译

`;r.Shaders.AllowCompilingThroughWorkers=0` 

这行注释掉，我们需要在Editor里调试ShaderCompile的过程

# Spir-V Shader 的编译入口

`Vulkan Shader`的编译入口处在

UE5_master\Engine\Source\Developer\VulkanShaderFormat\Private\VulkanShaderCompiler.cpp

`static bool CompileWithShaderConductor`

UE现在已经逐步废弃了老版的hlsl，而是走shaderconductor的编译流程。

当调试Shader编译错误的时候，可以在`FShaderConductorContext::CompileHlslToSpirv`这里下断点。

另外如果有额外的dxc的参数需要传也可以在这里魔改。



# DXIL Shader编译入口

UE5_master\Engine\Source\Developer\Windows\ShaderFormatD3D\Private\D3DShaderCompilerDXC.cpp

DXIL Shader可以考虑在 `static HRESULT D3DCompileToDxil(`这里下断点。



# DXBC Shader的编译入口(未验证)

ue\UE5_master\Engine\Source\Developer\Windows\ShaderFormatD3D\Private\D3DShaderCompiler.cpp
这个没仔细跟，但是可能下断点在`static bool CompileAndProcessD3DShaderFXCExt(`这里比较合适


使用FXC编译还是DXC编译是根据需要的 Shader Model等级确定的


```cpp
const bool bUseDXC =
	Language == ELanguage::SM6
	|| bIsRayTracingShader
	|| Input.Environment.CompilerFlags.Contains(CFLAG_WaveOperations)
	|| Input.Environment.CompilerFlags.Contains(CFLAG_ForceDXC)
	|| Input.Environment.CompilerFlags.Contains(CFLAG_InlineRayTracing);
```