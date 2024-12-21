
---
title: "UE | WPO Disable Distance的使用"
date: 2024-12-21T03:58:50Z
draft: false
categories: [ "UE"]
isCJKLanguage: true
slug: "af05ecba"
toc: true
mermaid: false
fancybox: false
blueprint: false
# latex support
# katex: true
# markup: mmark
# mmarktoc: false 
# UEVersion: 5.3.2 
---

这个功能在5.3才被彻底在普通管线上实现，在 5.2 之前这个功能是 nanite only 的。
可以根据距离停止WPO的计算，根据 Epic 某篇文章说在堡垒之夜有非常好的性能收益，一时之间找不到这个文章了，可能是这篇.
[Bringing Nanite to Fortnite Battle Royale in Chapter 4 - Unreal Engine](https://www.unrealengine.com/en-US/tech-blog/bringing-nanite-to-fortnite-battle-royale-in-chapter-4)

从原理上来说不复杂：
- 在 Shader 生成的时候，在计算 WPO 的shader 代码生成的时候外层包裹一个 If 条件
- 根据 GPUScene 上的剔除结果（逐Instance 剔除）来标记一个 flag，根据 flag 来确定是否跳过 WPO 计算


# UStaticMeshComponent::WorldPositionOffsetDisableDistance


经过层层传递，从U类到`StaticMeshProxy`类，再到`FPrimitiveUniformShaderParametersBuilder`


```cpp
void FPrimitiveSceneProxy::BuildUniformShaderParameters(FPrimitiveUniformShaderParametersBuilder &Builder) const{
...
float WPODisableDistance;
if (GetInstanceWorldPositionOffsetDisableDistance(WPODisableDistance))
{
    Builder.InstanceWorldPositionOffsetDisableDistance(WPODisableDistance);
}
...
}

FPrimitiveUniformShaderParametersBuilder& FPrimitiveUniformShaderParametersBuilder::InstanceWorldPositionOffsetDisableDistance(float WPODisableDistance)
{
	WPODisableDistance *= GetCachedScalabilityCVars().ViewDistanceScale;
	bHasWPODisableDistance = true;
	Parameters.InstanceWPODisableDistanceSquared = WPODisableDistance * WPODisableDistance;

	return *this;
}
```


非GPUScene的情况下, 注意这个数值通过UBO递给了Shader里的`FPrimitiveSceneData.InstanceWPODisableDistanceSquared`

GPUScene的情况下，数据通过`GPUScenePrimitiveSceneData`里读出来`(FPrimitiveSceneData GetPrimitiveData(uint PrimitiveId))`

# 这个数值的用处

WPO是在Vertex Shader里计算的，所以这个数值一定是在VS里屏蔽了VS的运算。


## Vertex Factory

shader里是不支持什么继承这种玩意的。
所以虚幻为了实现VertexFactory这种结构，采用的是基于Interface的设计。
**在Shader**里的ush里，不同VertexFactory需要定义一系列相同的结构。
在引擎编译的shader的时候，通过C++这边的组织关系拼接不同的ush上去，就可以实现链接上不同的VertexFactory。

使用VertexShaderFactory的主要流程:

- 调用GetVertexFactoryIntermediates(FVertexFactoryInput Input),通过Input和Uniform来获取这个VertexShaderFactory需要的各类数据和**状态**
- VertexShader里调用以下的各类函数来获取想要的数据


```cpp
VertexFactoryGetTangentToLocal(…)
VertexFactoryGetWorldPosition(…)
VertexFactoryGetRasterizedWorldPosition(…)
VertexFactoryGetPositionForVertexLighting(…)
VertexFactoryGetInterpolantsVSToPS(…)
```

![edit-62166f2624ac424aac9a490e2772dd1f-2024-08-03-16-58-28](https://img.blurredcode.com/img/edit-62166f2624ac424aac9a490e2772dd1f-2024-08-03-16-58-28.png?x-oss-process=style/compress)


## WPO的计算逻辑

从`LocalVertexShaderFactory`里的`GetVertexFactoryIntermediates`， 我们可以读到一些关于WPO的逻辑。

```cpp
#if !USE_INSTANCE_CULLING && USES_WORLD_POSITION_OFFSET && !ALWAYS_EVALUATE_WORLD_POSITION_OFFSET
	// In this case, we have to do the WPO disable distance check in the VS because it wasn't done in instance culling
	if (Intermediates.bEvaluateWorldPositionOffset) // 如果有Instance culling的情况，在Instance Culling的时候会逐Instance计算，这里只考虑单个StaticMeshComponet
	{
		const bool bPrimEvalWPO = (PrimitiveData.Flags & PRIMITIVE_SCENE_DATA_FLAG_EVALUATE_WORLD_POSITION_OFFSET) != 0;
		const bool bWPODisableDistance = (PrimitiveData.Flags & PRIMITIVE_SCENE_DATA_FLAG_WPO_DISABLE_DISTANCE) != 0;
		
		if (!bPrimEvalWPO || (bWPODisableDistance && InstanceViewDistSq >= PrimitiveData.InstanceWPODisableDistanceSquared)) // 这里就是判断是否超过 WPO Distance了
		{
			Intermediates.bEvaluateWorldPositionOffset = false;
		}
	}
#endif
```


第一行检查`PRIMITIVE_SCENE_DATA_FLAG_EVALUATE_WORLD_POSITION_OFFSET`，这个标记实际上是CPP这边设上去的，在
`Engine/Source/Runtime/Engine/Public/PrimitiveUniformShaderParametersBuilder.h` 这里,

`Parameters.Flags |= bEvaluateWorldPositionOffset ? PRIMITIVE_SCENE_DATA_FLAG_EVALUATE_WORLD_POSITION_OFFSET : 0u;`

默认是带这个标记的,但是在SceneProxy里也可以修改这个值，比如`StaticMeshSceneProxy`里就有对应的逻辑

`bEvaluateWorldPositionOffset = !IsOptimizedWPO() || InComponent->bEvaluateWorldPositionOffset;`



```cpp
// MobileBasePassVertex.usf
    // 将FLocalVertexIntermediates 转为 FMaterialVertexParameters, 数据上大体差不多
	FMaterialVertexParameters VertexParameters = GetMaterialVertexParameters(Input, VFIntermediates, WorldPosition.xyz, TangentToLocal); 
	float3 WorldPositionOffset = GetMaterialWorldPositionOffset(VertexParameters);
```


然后跳转到`MaterialTemplate.h`


```cpp
float3 GetMaterialWorldPositionOffsetRaw(FMaterialVertexParameters Parameters)
{
%s;  // 这里是MaterialEditor拼接代码的地方
}

float3 GetMaterialWorldPositionOffset(FMaterialVertexParameters Parameters)
{
	BRANCH
	if (ShouldEnableWorldPositionOffset(Parameters)) // 里面即是判断之前计算的 Intermediates.bEvaluateWorldPositionOffset 
	{
		return ClampWorldPositionOffset(Parameters, GetMaterialWorldPositionOffsetRaw(Parameters));
	}
	return float3(0, 0, 0);
}

```


# 如何使用 WPO Disable Disatnce?

- r.OptimizedWPO 0  永远evalute WPO
- ` UStaticMeshComponent::SetWorldPositionOffsetDisableDistance(int32 NewValue)` 设置新的剔除距离

# 实用性

这个对于StaticMesh等挺适合的，但是对于Foliage/ Grass有一定做法上的要求。
原因是Foliage / Grass 的最后几级LOD 通常会采用Billboard/Imposter来处理，而Billboard的旋转一般在材质里连WPO，如果直接禁用了WPO，Billboard的功能就破坏了。

这里可以考虑沿着两个方向改引擎:
1. 新增加一个WPO_Force引脚，WPO_Force引脚的计算不会受这个优化影响，这样可以区分开必须的WPO和效果向的WPO
2. Foliage不使用这个功能，而是改成在材质里实现Dynamic If，根据距离来跳过部分计算(如风场效果)

第一个方案改动比较小，但是第二个方案如果能正确搞定`Dynamic If`可以用在材质的很多地方，不局限在 WPO 上。
比如可以根据距离跳过一些多层纹理混色或者 RVT 采样等。

一个冷知识是虚幻在 5.0里也试图在材质里引入`If`, `For`等控制流相关的语句，后来还是弃坑了，代码的入口被删了(引擎里还是能翻到很多历史遗迹)，从 UDN 里看到说当时开发这个功能的老哥都跑路了，估计烂尾了。