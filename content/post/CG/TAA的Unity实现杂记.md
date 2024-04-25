
---
title: "Unity TAA实现杂记(附录带开箱即用代码)"
date: 2022-05-06T18:21:10+08:00
draft: false
# tags: [ "" ]
categories: [ "CG"]
# keywords: [ ""]
# lastmod: 2022-05-06T18:21:10+08:00
# CJKLanguage: Chinese, Japanese, Korean
isCJKLanguage: true
slug: "9f1444db"
toc: false
mermaid: false
# latex support
katex: true
markup: mmark
mmarktoc: true
---

# TAA

主要理论参考资料可以参考`Inside`的分享[GDC Vault - Temporal Reprojection Anti-Aliasing in INSIDE](https://www.gdcvault.com/play/1022970/Temporal-Reprojection-Anti-Aliasing-in)，主要实现代码可以参考Unity的[Post Processing v2的实现](https://github.com/Unity-Technologies/PostProcessing/blob/v2/PostProcessing/Runtime/Effects/TemporalAntialiasing.cs)，相较于Inside的实现其更加干净，而且更容易看懂。
## 框架
![edit-84c9ebe0655e44b9a7e08e01ff355f1f-2022-05-06-17-04-14](https://img.blurredcode.com/img/edit-84c9ebe0655e44b9a7e08e01ff355f1f-2022-05-06-17-04-14.png?x-oss-process=style/compress)

几个注意的点:
1. 输入的所有数据都是jitter后的
2. `unjitter`只发生在混合阶段，用以采样`_MainTex`，也即是jitter后的color buf。为了采样`unjitter`的数据，需要调整uv坐标。
3. `reproj`是找到当前帧的像素在之前帧的位置，有一些细节要处理(depth dilate)

## Jitter 视椎体

要注意的点是`Jitter`实际上是亚像素级别的轻微偏移裁剪近平面，形成`Temporal`上的超采样。

```C#
		_Jitter = new Vector2(
			2.0f * (HaltonSequence[Index].x - 0.5f) / camera.pixelWidth,
			2.0f * (HaltonSequence[Index].y - 0.5f) / camera.pixelHeight);
		_Jitter *= JitterScale;
		// Unity的矩阵是row-major
		// matrix[x,y]指的是x row, y col
		proj.m02 += _Jitter.x;
		proj.m12 += _Jitter.y;
```
把`_Jitter`分量放在`proj.m02`和`proj.m03`的位置，在齐次坐标系下其会成为`(_jitter.x * z_v) ,(_jitter.y * z_v)`,`z_v`是`view-space z`坐标。
经过透视除法后，z_v分量被消去。所剩下的在NDC坐标系下的偏移就是`_jitter.xy`，所以`_jitter.xy`设置为`[-1,1]`偏移就行。

## Motion Vectors的计算

`Motion Vectors`表示了同一个顶点在前后两帧中被渲染到`screenspace`的`uv`坐标之差。
不考虑jitter情况下，计算`Motion Vectors`可以划分为三种情况:
1. 相机不动，场景无运动物体: 不需要考虑`Motion Vectors`，加点jitter出来的结果直接颜色混合就行。
2. 镜头在动，场景不动：在冯乐乐的书籍里讨论了在这种场景下计算Motion Blur的方法。由于没有物体的运动，可以不通过额外的Pass，在屏幕空间利用cur_vp矩阵的逆矩阵从深度和uv重建世界坐标，并通过prev_vp矩阵计算上一帧的uv坐标，从而得到Motion Vector。
3. 镜头在动，场景在动：需要保存上一帧的mvp和这一帧的mvp，上一帧的uv信息可以通过上一帧MVP得到，当前帧的UV信息可以通过当前帧的MVP得到。需要额外一个Pass渲染整个场景的物体，以计算每个物体的Motion Vectors并写入到`R16G16_TYPELESS`纹理中。精细化的处理可以单独处理动态物体，以降低Overdraw。
   
写成伪代码可以写作
```
    newNDCPos = cur_frame_MVP * vertexPos;
    preNDCPos = prev_frame_MVP * vertexPos;
    new_uv = 0.5 * newNDCPos.xy + 0.5;
    pre_uv = 0.5 * preNDCPos.xy + 0.5;
    motion_vector = new_uv - pre_uv;
```

Unity的默认管线提供了`DepthTextureMode.MotionVectors`选项以帮助计算`Motion Vectors`，但是对于`Instance`的物体还是需要手动在Shader里计算Motion Vectors。
具体的代码实现可以看[Motion.cginc](https://github.com/BlurryLight/InstancedMotionVector/blob/4128fe5379ecd8734c5f65fe735bd25bfb11aae1/Assets/InstancedMotionVector/Motion.cginc#L20)
 
## Reprojection

采样Motion Vectors纹理获得`Motion Vectors`，即可获得上一帧的，也即在`_HistoryBuffer`纹理上的采样坐标。
伪代码可写作
```hlsl
float2 HistoryUV = i.texcoord - Motion;
float4 HistoryColor = _HistoryTex.Sample(sampler_LinearClamp, HistoryUV);
```

## 对抗Artifacts

### Ghosting
鬼影，又被称为history mismatch，是指在重投影的过程中(Reproj)，当前`pixel`的像素被重投影到上一帧的`color buffer`寻找其历史着色，但是由于几何关系遮挡等问题重投影的像素并非是这一个像素的历史像素，被称为`history mismatch`。

![edit-84c9ebe0655e44b9a7e08e01ff355f1f-2022-05-06-17-04-31](https://img.blurredcode.com/img/edit-84c9ebe0655e44b9a7e08e01ff355f1f-2022-05-06-17-04-31.png?x-oss-process=style/compress)

对抗`Ghosting`主要靠检测颜色，如果一个像素重投影采样历史帧的颜色和当前帧的颜色相差很大，可以充分认为发生了`history mismatch`。
一种可行的方式是采样当前像素在当前帧周围的 $$ 3\times3 $$ 邻居的像素的颜色值，计算一个最大的颜色`AABB`包围盒。这个所谓的**颜色**可以选用在不同的色彩空间，比如RGB,YCoCg等不同的颜色空间做。对于在包围盒以外的点，也即是发生了`history mismatch`的像素，有`clamp`和`clip`两种不同的处理方式。
![edit-84c9ebe0655e44b9a7e08e01ff355f1f-2022-05-06-17-52-18](https://img.blurredcode.com/img/edit-84c9ebe0655e44b9a7e08e01ff355f1f-2022-05-06-17-52-18.png?x-oss-process=style/compress)

Neighborhood clamping也有他的问题，比如如这篇文章里描述的场景[TAA Ghosting 的相关问题](https://www.cnblogs.com/crazii/p/7244300.html)，在相邻像素差别很大的情况(比如一个白色`1,1,1`，一个黑色`0,0,0`)下所计算的颜色包围盒可能相当大，此时裁剪完全失效。

同时，对于`history mismatch`的处理方式也是值得考量的。
考虑极端情况下，一律接受`history`，那么就是鬼影加上画面变糊，一律拒绝`history`，那么就是没有TAA抗锯齿的效果。
因此`AABB`画的越大，画面就会越糊，`AABB`越小，比如Nvidia提出的`Variance Clip`，越容易拒绝历史帧的颜色，走样就会冒出来。

#### YCoCg空间AABB

颜色空间是一个三维空间，选取不同的基函数，可以以不同的形式表示相同的空间。
UE认为同一个物体表面附近的像素在色调上往往类似，只是着色上亮度有较大差异，想了下`diffuse`表面的物体好像差不多是这个情况。
`YCoCg`颜色空间有一维是亮度`luma`，因此在`YCoCg`下做计算AABB，转换到RGB空间下做可视化可以发现得到的包围盒比较像有向包围盒，其包围盒有一维是沿着亮度方向的，其AABB会更窄。

![TAA的Unity实现杂记-2022-05-08-00-41-32](https://img.blurredcode.com/img/TAA的Unity实现杂记-2022-05-08-00-41-32.png?x-oss-process=style/compress)

三维情况下的RGB空间的AABB和`YCoCg`空间的AABB对比可见图，具体见附录:

{{< figure src="https://img.blurredcode.com/img/TAA的Unity实现杂记-2022-05-21-00-05-32.png" width="50%" caption="黑色为RGB AABB，蓝色为YCoCg AABB">}}

![aabb comparison](https://img.blurredcode.com/img/202205211438006.gif)
#### Variance Clip

`Nvidia`的[GDC分享](https://developer.download.nvidia.com/gameworks/events/GDC2016/msalvi_temporal_supersampling.pdf)里从正态分布的角度出发，其不是直接计算周围9个像素点颜色的AABB。
而是先用这九个像素点作为样本，估计一个正态分布的期望$$\mu$$和标准差$$\sigma$$。
并将AABB的最小值和最大值确立为$$ \mu - \gamma \sigma$$，$$ \mu + \gamma \sigma$$，其中$$ \gamma $$是一个默认值为1的超参数，通过人为调节$$\gamma$$可以调整AABB的大小。

在原来的情况下，如果周围的邻居有一个亮点，那么AABB会被画的特别大。
但是在`variance clip`这种正态分布的模型下，单个离群像素点的影响被降低了，所以可以得到更小的AABB。



{{< imgCompare 
	ImgWidth="40" 
	ImgSrc0="https://img.blurredcode.com/img/TAA的Unity实现杂记-2022-05-08-00-53-41.png?x-oss-process=style/compress" 	ImgAlt0="NoTAA" 
	ImgSrc1="https://img.blurredcode.com/img/TAA的Unity实现杂记-2022-05-08-00-54-00.png?x-oss-process=style/compress" 	ImgAlt1="TAA" 
	ImgCaption="左:Raw AABB 右: Variance Clip">}}

写成伪代码大致可以写作
```hlsl
float3 m1 = 0,m2 = 0;
for (int k = 0; k < 9; k++)
{
	float3 C = RGBToYCoCg(_MainTex.Sample(sampler_PointClamp, uv, kOffsets3x3[k]));
	m1 += C;
	m2 += C * C;
}

float3 mu = m1 / 9;
// sigma的计算公式严格来说不是这样的
//https://en.wikipedia.org/wiki/Standard_deviation，这里是一个近似
float3 sigma = sqrt(abs(m2 / 9 - mu * mu));
#define GAMMA 1.0f

AABBMin = mu - GAMMA * sigma;
AABBMax = mu + GAMMA * sigma;
```
### 走样
#### 边缘几何走样

由于`Motion Vectors`图也是有锯齿的，所以直接用中心点采样的方式在边缘处抗锯齿会失效。
一种保守的策略是选取$$3\times3$$区域内的深度最小的点，这样可以确保在几何边缘的像素点能采样到`Motion Vector`，从而正确采样history颜色。
![edit-84c9ebe0655e44b9a7e08e01ff355f1f-2022-05-06-18-02-20](https://img.blurredcode.com/img/edit-84c9ebe0655e44b9a7e08e01ff355f1f-2022-05-06-18-02-20.png?x-oss-process=style/compress)

#### 内部几何走样

这个问题比较复杂，往往是由于小三角形引起的。比如远处的树叶等小三角形，其大小甚至小于一个像素，这会导致在相机jitter过程中该三角形一会出现一会不出现。
灵魂画师画个图，方框代表一个像素。在`jitter`过程中，采样点不一定能采样到这个三角形。
![TAA的Unity实现杂记-2022-05-06-19-00-01](https://img.blurredcode.com/img/TAA的Unity实现杂记-2022-05-06-19-00-01.png?x-oss-process=style/compress)

这一问题的解决方式在这篇文章中得到了讨论(https://zhuanlan.zhihu.com/p/71173025)。

#### 着色高光走样

这个问题的成因感觉还没完全想清楚，但是大概和上一个的原因差不多。
一些零碎的点状的高光，在相机抖动的过程中，可能一帧高光被采样到，一帧没有。这样会导致`AABB Clip`反复发生，从而使得该像素不能稳定的和历史帧的颜色混合，呈现出高光闪烁的特点。
要压制这个问题可以考虑做一次filter以压制这种高光，不过会导致画面变糊。
其他的方式可以参考这篇文章(https://zhuanlan.zhihu.com/p/64993622)。

### 画面变模糊
由于在采样历史帧的时候采用`Linear Sample`可能导致画面，尤其是几何边缘变糊。
`Unity`在TAA处理中还加入了一个锐化来防止画面变糊。

其代码大致可以写成
```hlsl
float2 uv = i.texcoord - _Jitter;
float4 Color = _MainTex.Sample(sampler_LinearClamp, uv);
float4 topLeft = _MainTex.Sample(sampler_LinearClamp, uv - _MainTex_TexelSize.xy * 0.5);
float4 bottomRight = _MainTex.Sample(sampler_LinearClamp, uv + _MainTex_TexelSize.xy * 0.5);
float4 corners = 4.0 * (topLeft + bottomRight) - 2.0 * Color;
// Sharpen output
//这里实际上是一个这样的核,0.166667是1/6，2.718是自然对数
    /*              | -(2/3)*x                          |      |topLeft     |
	*   Color = |          ((4/3)x + 1)             | *    |Color       |
	*           |                          -(2/3)x  |      |bottomRight |
	*  其中 x = e * _Sharpness。通过_Sharpness参数控制锐化核的程度
	*/
Color = Color + (Color - (corners * 0.166667)) * 2.718282 * _Sharpness;

// ...
// do blending with Color and HistoryColor
```

# 结果
{{< imgCompare 
	ImgWidth="100" 
	ImgSrc0="https://img.blurredcode.com/img/TAA的Unity实现杂记-2022-05-06-19-18-14.png?x-oss-process=style/compress"
	ImgSrc1="https://img.blurredcode.com/img/TAA的Unity实现杂记-2022-05-06-19-17-58.png?x-oss-process=style/compress"
	ImgCaption="左:NO TAA 右: TAA">}}
# Reference

1. [TAA原理与OpenGL实现 - Irimsky](https://www.irimsky.top/archives/301/)
2. [Temporal AA Anti-Flicker](https://zhuanlan.zhihu.com/p/71173025)
3. [DX12渲染管线(2) - 时间性抗锯齿(TAA)](https://zhuanlan.zhihu.com/p/64993622)
4. 以及文章里出现的其他引用


# Appendix

## 代码

[Unity Builtin Pipeline Simple TAA](https://gist.github.com/BlurryLight/145131dbacac34345908c529a3488e8f)

## AABB的比较
<script src="https://gist.github.com/BlurryLight/70e2c778d912996901a0a9d0e3ce18f5.js"></script>