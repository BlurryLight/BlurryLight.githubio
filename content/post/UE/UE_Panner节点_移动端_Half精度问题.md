
---
title: "UE Panner节点移动端Half精度问题"
date: 2024-08-11T19:31:32+08:00
draft: false
categories: [ "UE"]
isCJKLanguage: true
slug: "3ffbfa69"
toc: true
mermaid: false
fancybox: false
blueprint: false
# latex support
# katex: true
# markup: mmark
# mmarktoc: false 
UEVersion: 5.3.2 
---


Panner节点主要是用于实现材质中纹理的平滑滚动的效果，可以用在水花等地方。
最近发现在移动端上，随着时间的推移，用了该节点的材质会滚动得越来越慢，越来越卡顿，应该是精度问题。

# Panner节点的实现

panner节点的实现位于 `UMaterialExpressionPanner::Compile`，它自身并没有什么特殊的HLSL代码，只是组合了一些基本算子。

勾上了bFractionalPart:
- `UV = UV + frac(Time * Speed)`

没有勾上bFractionalPart:
- `UV = UV + Time * Speed`


# 问题的源头

![UE_Panner节点_移动端_Half精度问题-2024-08-11-19-37-45](https://img.blurredcode.com/img/UE_Panner节点_移动端_Half精度问题-2024-08-11-19-37-45.png?x-oss-process=style/compress)

连一个最简单的材质，直接打开`DirectX Mobile`来看翻译出来的hlsl


```cpp
	half Local1 = (View_GameTime * 0.94999999);
	half Local2 = frac(Local1);

	half2 Local3 = Parameters.TexCoords[0].xy;
	half2 Local4 = half2(  Local2 ,frac(0.00000000));
	half2 Local5 = (  Local4  +   Local3 );
	half Local6 =  1.0f;
	half4 Local7 = ProcessMaterialColorTextureLookup(Texture2DSample(Material_Texture2D_0,Material_Texture2D_0Sampler,  Local5 ));
	half Local8 =  1.0f;
```

问题出在`Panner`节点的翻译上，其并不是将 `frac(GameTime * Speed)` 这个表达式翻译成一个inline语句，而是先暂存了一个中间变量，然后再做frac。
这样，` half Local1 = (View_GameTime * 0.94999999); ` 这一句中将 `float32` 存到一个half变量中，就会丢失精度，后面再做frac已经是对丢失精度变量以后的结果做的了。

# 解决方案

1. 如果不想改引擎，可以直接在移动端上禁用Panner节点(参考SceneColor节点如何屏蔽ES31），然后项目组写一个Custom节点来处理UV的计算。
2. 如果要改引擎，改法也可以有几种:

## **比较系统的方案**:

最系统的方案是实现一个 `Float Scope`，可以将材质里部分连接的节点提升到float32计算，这样就一劳永逸解决很多问题。
我参考了以下这篇好文修改了一番材质编辑器，在实现Dynamic Branch的时候顺便也把Float Scope给做了。

> 参考：[UE4实现动态分支及相关材质节点编译原理 - 知乎](https://zhuanlan.zhihu.com/p/563820416)

大体上思路是一样的，需要额外在编译的时候追踪哪些节点位于某个`Scope`内，然后对这个Scope内的节点在生成HLSL的时候做额外的处理。

材质里面编译出来的数据类型都是`MaterialFloat`，然后平台根据宏来将这个类型转换成`half`或者`float`。
我们只需要在FloatScope开始和结束的时候重新定义`MaterialFloat`的类型，就可以做到自由控制某一段材质节点的计算精度。

修改后的Panner节点生成的hlsl如下，可以看到在计算UV时候被转到了`float`计算
```cpp

                    #undef MaterialFloat 
                    #undef MaterialFloat2 
                    #undef MaterialFloat3 
                    #undef MaterialFloat4
                    #undef MaterialFloat3x3 
                    #undef MaterialFloat4x4 
                    #undef MaterialFloat4x3 
                    
                    #define MaterialFloat float
                    #define MaterialFloat2 float2
                    #define MaterialFloat3 float3
                    #define MaterialFloat4 float4
                    #define MaterialFloat3x3 float3x3
                    #define MaterialFloat4x4 float4x4 
                    #define MaterialFloat4x3 float4x3 
    MaterialFloat Local1 = (View.GameTime * 0.94999999);
    FloatDeriv Local2 = FracDeriv(ConstructConstantFloatDeriv(Local1));

                    #undef MaterialFloat 
                    #undef MaterialFloat2 
                    #undef MaterialFloat3 
                    #undef MaterialFloat4 
                    #undef MaterialFloat3x3 
                    #undef MaterialFloat4x4 
                    #undef MaterialFloat4x3 

                    #if PIXELSHADER && !FORCE_MATERIAL_FLOAT_FULL_PRECISION
                        #define MaterialFloat half
                        #define MaterialFloat2 half2
                        #define MaterialFloat3 half3
                        #define MaterialFloat4 half4
                        #define MaterialFloat3x3 half3x3
                        #define MaterialFloat4x4 half4x4 
                        #define MaterialFloat4x3 half4x3 
                    #else
                        // Material translated vertex shader code always uses floats, 
                        // Because it's used for things like world position and UVs
                        #define MaterialFloat float
                        #define MaterialFloat2 float2
                        #define MaterialFloat3 float3
                        #define MaterialFloat4 float4
                        #define MaterialFloat3x3 float3x3
                        #define MaterialFloat4x4 float4x4 
                        #define MaterialFloat4x3 float4x3 
                    #endif
                    
    FloatDeriv2 Local3 = ConstructFloatDeriv2(Parameters.TexCoords[0].xy,Parameters.TexCoords_DDX[0].xy,Parameters.TexCoords_DDY[0].xy);
    FloatDeriv2 Local4 = ConstructFloatDeriv2(MaterialFloat2(DERIV_BASE_VALUE(Local2),frac(0.00000000)),MaterialFloat2(Local2.Ddx, 0.0f),MaterialFloat2(Local2.Ddy, 0.0f));
    FloatDeriv2 Local5 = AddDeriv(Local4,Local3);
```



## **比较简易的方案**:(我没尝试，但是应该行得通)
我们知道这里产生的Bug的源头是这里产生了一个中间变量。
回到代码里去看可以发现是这一部分

```cpp
		Arg1 = Compiler->PeriodicHint(Compiler->Frac(Compiler->Mul(TimeArg, SpeedXArg)));
		Arg2 = Compiler->PeriodicHint(Compiler->Frac(Compiler->Mul(TimeArg, SpeedYArg)));
```

调到`Compiler->Mul`的实现,除去一大堆关于`Uniform/Constant`的特殊优化分支以外，它最后实际是走到了

```cpp
    return AddCodeChunk(GetArithmeticResultType(A,B),TEXT("(%s * %s)"),*GetParameterCode(A),*GetParameterCode(B));
```

这里`AddCodeChunk`会产生一个中间的变量来暂存表达式的结果。

可以考虑添加一个`Compiler->inlineMul`函数，最后调用`AddInlinedCodeChunk`这个变体，就应该可以直接生成`frac(Time * Speed)`这样的代码了。
