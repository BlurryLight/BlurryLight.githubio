
---
title: "UE5 | 一个Mobile小于4x4的法线贴图采样错误的问题"
date: 2024-07-05T23:44:10+08:00
draft: false
categories: [ "UE"]
isCJKLanguage: true
slug: "59e1920f"
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



最近注意到在移动端上Build出来的HLOD看上去光照不太对，明显感觉法线出了点问题。但是在PIE和PC包里看上去都没问题。
把Android上的Shader抓出来，起了一个Vulkan-RHI的Editor调到Mobile Render，然后抓帧把Shader打出来一行一行的diff..
翻了一下代码以后发现两边对 Normap Map 的采样处理的不太一样


```cpp
 */
MaterialFloat4 UnpackNormalMap( MaterialFloat4 TextureSample )
{
	#if DXT5_NORMALMAPS   // DirectX 
		MaterialFloat2 NormalXY = TextureSample.ag;
	#elif LA_NORMALMAPS   // ASTC..
		MaterialFloat2 NormalXY = TextureSample.ra;
	#else
		MaterialFloat2 NormalXY = TextureSample.rg;
	#endif

	NormalXY = NormalXY * MaterialFloat2(2.0f,2.0f) - MaterialFloat2(1.0f,1.0f);
	MaterialFloat NormalZ = sqrt( saturate( 1.0f - dot( NormalXY, NormalXY ) ) );
	return MaterialFloat4( NormalXY.xy, NormalZ, 1.0f );
}
```

最后发现在PIE下采的是`TextureSample.rg`而在Android上采的是`TextureSample.ra`。

# 特殊的法线贴图

如果HLOD生成的时候不勾选烘焙法线，默认的基础材质会填充一个`1x1`的DummyNormalMap上去。

![UE_5_3_ASTC_NormalMap渲染错误的问题-2024-07-05-23-48-50](https://img.blurredcode.com/img/UE_5_3_ASTC_NormalMap渲染错误的问题-2024-07-05-23-48-50.png?x-oss-process=style/compress)

可能是这个问题，沿着这个思路，连了一个很基础的材质，

![UE_5_3_ASTC_NormalMap渲染错误的问题-2024-07-05-23-49-14](https://img.blurredcode.com/img/UE_5_3_ASTC_NormalMap渲染错误的问题-2024-07-05-23-49-14.png?x-oss-process=style/compress)


打了个包看了下，果然能复现问题.. Android上采样的结果是错误的。

![UE_5_3_ASTC_NormalMap渲染错误的问题-2024-07-05-23-49-53](https://img.blurredcode.com/img/UE_5_3_ASTC_NormalMap渲染错误的问题-2024-07-05-23-49-53.png?x-oss-process=style/compress)


# 问题原因

关键词 `LA_NORMALMAPS`

阅读了一下代码，虚幻在编译移动端Shader时，如果发现当前配置使用了`ARM ASTC Encoder`时，会将BC5压缩格式映射到`ASTC_NORMALLA`，并且定义`LA_NORMALMAPS`这个宏(会改变Shader里采样Normal Map的Swizzling)。

```cpp
	if (bSupportsNormalLA && TextureFormatName == AndroidTexFormat::NameBC5)
		{
			TextureFormatName = AndroidTexFormat::NameASTC_NormalLA;
			continue;
		}
```

具体这个配置有啥特殊的可以参考 ARM_ASTC_ENCODER的文档，`https://github.com/ARM-software/astc-encoder/blob/main/Docs/Encoding.md#equivalence-with-other-formats`，可能对精度保持的更好一点?

然后实际上，如果我们使用低于`4x4`的`RGBA8`法线贴图，那么这张纹理由于低于ASTC最低可压缩的blockSize，所以虚幻在Cook的时候会跳过压缩这张纹理。

所以这里出现了Shader里采样的通道实际上对不上贴图的通道的问题，贴图只有`RG`有值，而实际采样的是`RA`通道。

定位到问题就好解决了，不要使用低于`4x4`的法线贴图。最好在 Cook Texture的时候检查这种例子，拦截这种情况。