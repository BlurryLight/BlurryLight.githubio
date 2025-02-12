
---
title: "Mobile | 通过UAV和RTV Clear纹理的测试"
date: 2025-02-12T02:12:58Z
draft: false
categories: [ "UE"]
isCJKLanguage: true
slug: "e20059a6"
toc: false
mermaid: false
fancybox: false
blueprint: false
# latex support
# katex: true
# markup: mmark
# mmarktoc: false 
# UEVersion: 5.3.2 
---

# Clear By UAV / RTV

高通845 XIAOMi 8

PF_RGBA8

|Res|UAV|RTV|
|-|-|-|
|64x64|14us|11us|
|256|37|21|
|512|116|52|
|1024|443|170|
|2048|1698|621|
|4096|6.7ms|2.4ms|
|8192|24.9ms|10.08ms|


测试代码
测试数据用Snapdragon Profiler抓出来的

```cpp
void FShaderDemoModule::RandomGraphicsTest_Renderthread(FPostOpaqueRenderParameters& Parameters)
{
	FRDGBuilder* RDGBuilderPtr = Parameters.GraphBuilder;
	static const std::array TextureSizes{64, 256, 512, 1024, 2048, 4096, 8192};
	auto ClearTest = [RDGBuilderPtr](bool bClearByUAV)
	{
		auto&& RDGBuilder = *RDGBuilderPtr;
		RDG_EVENT_SCOPE(RDGBuilder, "%d ClearTest", bClearByUAV);
		for(auto Size : TextureSizes)
		{
			FRDGTextureDesc Desc = FRDGTextureDesc::Create2D(FIntPoint(Size, Size), PF_R8G8B8A8, FClearValueBinding::Black,
															 TexCreate_RenderTargetable | TexCreate_UAV);
			FRDGTextureRef Texture = RDGBuilder.CreateTexture(Desc,TEXT("RandomGraphicsTest"));
			FRDGTextureUAVRef UAV = RDGBuilder.CreateUAV(Texture);
			AddClearUAVPass(RDGBuilder, UAV, FUintVector4(1, 1, 1, 1), ERDGPassFlags::Compute | ERDGPassFlags::NeverCull);
			AddClearRenderTargetPass(RDGBuilder, Texture);
			RDGBuilder.ConvertToExternalTexture(Texture); // avoid culling
		}
	};
	ClearTest(0);
}
```

# 结论：

除非特殊需求，不然用硬件自带的ClearTexture更快。 至少在低端手机上是这样。
而且带宽更友好一点。(CS没有透明压缩)