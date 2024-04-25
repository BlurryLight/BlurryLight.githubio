
---
title: "UE5 Strange Magenta Pixels Sampled in ASTC6x6 On Android"
date: 2024-04-25T23:52:24+08:00
draft: false
categories: [ "UE" ]
isCJKLanguage: true
slug: "ea7c59b3"
toc: false
mermaid: false
fancybox: false
blueprint: false
# latex support
# katex: true
# markup: mmark
# mmarktoc: false 
---

{{% notice info %}}
    Engine Version: 5.3.2
{{% /notice %}}


Recently I noticed there are some strange magenta pixels `(1,0,1,1)` appeared on objects on Android Package.
After quick investigation through Renderdoc, I noticed some textures compressed in ASTC6x6 format have there mipmaps lower than 4x4 filled with magenta pixels.

I'm not sure which way is correct to handle the 4x4 mipmaps and lower for ASTC6x6. 
These low mipmap grades should probably be filled with the correct pixels (but still take up 6x6 space), or simply not generate these mipmap levels.
But in no ways they should be filled with magenta pixels.


# Quick Solution

There are two ASTC encoder used in Unreal Engine, one is `astcenc` provided by Arm and the other is `ISPCTextureCompressor` provided by Intel.
Currently we are using ``ISPCTextureCompressor`.

I found a similar case reported by other developers in Epic UDN forum and the solution is to use `astcenc` instead of `ISPCTextureCompressor` to solve the problem.
One of Epic staff replied `astcenc` is now the default and recommended ASTC encoder and `ISPCTextureCompressor` support in Unreal is deprecated and no longer maintained.


```cpp
// when GASTCCompressor == 0 ,use TextureFormatIntelISPCTexComp instead of this
// @todo Oodle : GASTCCompressor global breaks DDC2.  Need to pass through so TBW can see.
int32 GASTCCompressor = 1;
static FAutoConsoleVariableRef CVarASTCCompressor(
	TEXT("cook.ASTCTextureCompressor"),
	GASTCCompressor,
	TEXT("0: IntelISPC, 1: Arm"),
	ECVF_Default | ECVF_ReadOnly
);
```


# An interesting history of astcenc in UE

The solution reminds me of an interesting background.
In the early days of Unreal Engine 4, `ISTCTextureCompressor` was the default ASTC encoder used in Unreal Engine.
Although it has some obvious issues (it doesn't support `10x10` and `12x12`), it was still the default encoder for a long time.
The main reason, I guess, is that Epic guys grabbed a wrong version of `astcenc` and got some misleading benchmark results.

In UE4.26, a very old version of `astcenc` (1.3) was bundled in Unreal Engine. 
It was built in about 2013, which was a 32bit executable and without any SIMD optimization.

![UE_532_ASTC6x6_Sample_Magenta_Pixels_On_Android-2024-04-26-00-17-38](https://img.blurredcode.com/img/UE_532_ASTC6x6_Sample_Magenta_Pixels_On_Android-2024-04-26-00-17-38.png?x-oss-process=style/compress)

So, it's not surprising that `ISPCTextureCompressor` completely crushes astcenc in terms of speed.

If you are still using UE4.26 or earlier, you may want to grab a latest `astcenc3` from internet and replace the old one.
We ran into this problem perhaps also because some configurations were forgotten in the config file during the upgrade of the engine.