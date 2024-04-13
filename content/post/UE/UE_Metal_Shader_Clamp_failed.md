
---
title: "UE半精度下SpirV-Cross生成了错误的Metal Shader"
date: 2024-04-13T14:55:20+08:00
draft: false
categories: [ "UE"]
isCJKLanguage: true
slug: "a86c2360"
toc: true
mermaid: false
fancybox: false
blueprint: false
# latex support
# katex: true
# markup: mmark
# mmarktoc: false 
---

# UE5 Metal后端打开半精度Shader编译支持

虚幻默认在IOS上是禁用半精度的，但是实际上手机上需要这个功能。
禁用的地方在于 `BaseEngine.ini`里

```ini
[/Script/IOSRuntimeSettings.IOSRuntimeSettings]
ForceFloats=True 
```

在游戏里的`DefaultEngine.ini`找个地方重新覆盖这个变量为false即可打开半精度支持。


# SpirV-Cross生成了错误的Metal Shader

打开半精度后发现有部分材质有 `error: call to 'clamp' is ambiguous`的编译错误，仔细调查后发现问题在于SpirV-Cross生成的MSL有问题。

虚幻编译MSL的基本流程是通过`ShaderConductor`调用 `hlsl --(dxc)-> spirv -- (spirv-cross) --> msl`.

通过简单的二分查找，发现这个问题出现在我们TA连的一些材质里会使用了 `arctangent2`这个节点，后面经过一系列的计算然后最后经过`saturate`就必然出现这个问题。

找了个最简单的材质连了一下，并且把`spir-v`打印出来，发现生成的spir-v是没有问题的，正确的处理了half精度。
那么只能开始怀疑是`spirv-cross`的问题。


```spir-v
%15 = OpExtInst %half %1 Atan2 %half_0x1p_0 %half_0x1p_1
%16 = OpExtInst %half %1 FClamp %15 %half_0x0p_0 %half_0x1p_0
```

## 最简单可复现例子

用`hlsl`写了一个最简单的例子: https://godbolt.org/z/95axs5xWY 

通过`dxc -T ps_6_6 -E PSMain -spirv  -enable-16bit-types `进行编译。

```hlsl
float4 PSMain(PSInput input) : SV_Target0
{
  return saturate( atan2(1.0h, 2.0h));
}
```

经过dxc编译 + spir-v编译以后最后确定生成了有问题的msl (https://shader-playground.timjones.io/85e3977ba4553f118bd37e67b325852c)

```c
out.out_var_SV_Target0 = float4(float(clamp(precise::atan2(half(1.0), half(2.0)), half(0.0), half(1.0))));
```

# 问题定位

问题现在确定在`spirv-cross`里了，打开他的代码库翻了一下，以`precise::atan2`为关键词搜索了一下，果然发现了可能是问题原因的代码:

https://github.com/KhronosGroup/SPIRV-Cross/blob/06407561ece7d7e78544112189f3eee13adf9959/spirv_msl.cpp#L10415-L10416

```c
// Override for MSL-specific extension syntax instructions.
// In some cases, deliberately select either the fast or precise versions of the MSL functions to match Vulkan math precision results.
void CompilerMSL::emit_glsl_op(uint32_t result_type, uint32_t id, uint32_t eop, const uint32_t *args, uint32_t count)
{
	case GLSLstd450Atan2:
		emit_binary_func_op(result_type, id, args[0], args[1], "precise::atan2");
		break;
```

通过这个函数的注释可以看到，在生成`msl`的时候，对部分函数需要特殊处理`fast`或者`precise`版本，而`atan2`是其中之一，这里默认选择了`precise`版本，也就是无论`spir-v`里是`half`还是`float`, 都会生成`precise::atan2`。

所以这会导致我们生成一个 `clamp(float,half,half)`这样的调用，MSL不支持这种混合精度的重载从而导致编译报错。

# 解决方案


[MSL: invalid code generated when atan2 and saturate are used in mixed precision environment · Issue #2309 · KhronosGroup/SPIRV-Cross](https://github.com/KhronosGroup/SPIRV-Cross/issues/2309)

首先给他提了一个issue，看Khronos那边怎么处理这个情况..但是就算khronos那边修复了，也要等虚幻下次合并spir-v cross才能用上了..

临时方案可以用虚幻的节点`Arctangent2Fast`绕过去。
具体的实现在`fastMath.ush`里，这个函数的拟合来自 https://seblagarde.wordpress.com/2014/12/01/inverse-trigonometric-functions-gpu-optimization-for-amd-gcn-architecture/， 用mathmatica拟合的结果。

误差分析和渲染测试可以看原文，我在虚幻里测试了基本没有肉眼可见的差异。

```hlsl
float atan2Fast( float y, float x )
{
	float t0 = max( abs(x), abs(y) );
	float t1 = min( abs(x), abs(y) );
	float t3 = t1 / t0;
	float t4 = t3 * t3;

	// Same polynomial as atanFastPos
	t0 =         + 0.0872929;
	t0 = t0 * t4 - 0.301895;
	t0 = t0 * t4 + 1.0;
	t3 = t0 * t3;

	t3 = abs(y) > abs(x) ? (0.5 * PI) - t3 : t3;
	t3 = x < 0 ? PI - t3 : t3;
	t3 = y < 0 ? -t3 : t3;

	return t3;
}
```