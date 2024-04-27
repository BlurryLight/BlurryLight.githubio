
---
title: "一个有意思的UE SwitchParam Dynamic Branch 导致驱动崩溃/渲染错误问题"
date: 2024-04-25T00:10:18+08:00
draft: false
categories: [ "UE"]
isCJKLanguage: true
slug: "79b3d5d0"
toc: false
mermaid: false
fancybox: false
blueprint: false
# latex support
# katex: true
# markup: mmark
# mmarktoc: false 
---

UE在5.2(也许5.3)推出的一个功能，将材质里的SwitchParam变成了Dynamic Branch(但是不支持动态修改，只允许在MI处修改)，主要目的是充分利用现代GPU对于Uniform Variable + Branch的优化，对于Uniform变量作为if condition的情况下，动态分支并不会带来warp divergence。
而StaticSwitch的问题在于会导致材质变体指数级上升，尤其是切换Shader变体可能会导致PSO的重新编译和加载，开销可能远大于Shader里走一个动态分支。

![UE_StaticSwitch_Dynamic_Branch_导致驱动bug-2024-04-25-00-12-49](https://img.blurredcode.com/img/UE_StaticSwitch_Dynamic_Branch_导致驱动bug-2024-04-25-00-12-49.png?x-oss-process=style/compress)

# Dynamic Branch的实现

Dynamic Branch的代码生成是在`int32 FHLSLMaterialTranslator::DynamicBranch(int32 Condition, int32 A, int32 B)`这个函数里，生成的代码类似于

```cpp

float3 branch1 = ...;
float3 branch2 = ...;

float3 staticResult;
[branch] switch (condition){ default: staticResult = branch1; break; case 0: staticResult = branch2; break;}
```

shader代码看起来似乎要先把两个分支的结果计算出来，然后再根据condition来选择，但是我怀疑驱动在执行的时候会直接跳过不需要的计算(因为Uniform此时是已知的)。

# 一个Vulkan上的Bug

在项目里实际用的时候，发现Dynamic Branch会在特定情况下会导致驱动崩溃，或者出现渲染结果错误:

- 预览等级为Es31, Editor下可以开到Android Vulkan Mobile
- 渲染API为Vulkan (Editor下不能用DirectX模拟)
- dynamic branch两侧出现了同一个TextureObj

![UE_StaticSwitch_Dynamic_Branch_导致驱动bug-2024-04-25-00-23-04](https://img.blurredcode.com/img/UE_StaticSwitch_Dynamic_Branch_导致驱动bug-2024-04-25-00-23-04.png?x-oss-process=style/compress)


这种情况下光看生成的代码是看不出来具体的问题的，

```c
	half2 Local0 = Parameters.TexCoords[0].xy;
	half Local1 =  1.0f;
	half4 Local2 = ProcessMaterialColorTextureLookup(Texture2DSample(Material_Texture2D_0,Material_Texture2D_0Sampler,  Local0 ));
	half Local3 =  1.0f;
	half3 Local4 = lerp(Local2.rgb,((half3)1.00000000),0.20000000);

half3 Static5;
[branch] switch (int(UnpackUniform_bool(asuint(Material_PreshaderBuffer[1][0]), 0))){ default: Static5 = Local2.rgb; break; case 0: Static5 = Local4; break;}
	half3 Local6 = Static5;
	half3 Local7 = lerp(Local6,Material_PreshaderBuffer[2].xyz,Material_PreshaderBuffer[1].y);

	PixelMaterialInputs.EmissiveColor = Local7;
```

spir-v我也仔细核对了一下，没什么问题， 注意这里开了优化以后，原来的代码里的`[branch] switch`直接被优化成了 `OpSelect`而不是我想的`OpBranchConditional`，不过本质上是等价的。

```
%67 = OpAccessChain %_ptr_Uniform_float %Material %int_0 %int_0 %uint_0
%68 = OpLoad %float %67
%69 = OpBitcast %uint %68
%70 = OpShiftRightLogical %uint %69 %uint_0
%71 = OpBitwiseAnd %uint %70 %uint_1
%72 = OpINotEqual %bool %71 %uint_0
%73 = OpSelect %int %72 %int_1 %int_0
%74 = OpIEqual %bool %73 %int_0
%75 = OpCompositeConstruct %v3bool %74 %74 %74
%22 = OpSelect %v3float %75 %21 %83
```


以上的代码看起来没啥问题，但是在实际运行的时候，会出现驱动崩溃，或者渲染结果错误的情况。我甚至拿RenderDoc对着Spir-v一行一行的调试，最后renderdoc调试器显示的结果和实际framebuffer的存在的颜色都对不上。


- 在8Gen1 Android13上渲染效果有问题，混入了不属于贴图的颜色
- 在公司的RTX3060上渲染效果有问题，显示为彩虹色
- 在另一台RTX3070的机器上 + 546.12驱动会导致Vulkan创建PSO失败，失败原因是Nvidia驱动崩溃， `NVVM compilation failed: 1`
- 升级驱动到552.22，渲染正常。
  

# 结论

Dynamic Branch 功能在特定情况可能会导致驱动bug，可以在vulkan下禁用这个功能，还是走静态分支编译成变体。