
---
title: "BRDF | 白炉测试"
date: 2024-12-08T08:39:56Z
draft: false
categories: [ "CG", "PBRT"]
isCJKLanguage: true
slug: "4d821343"
toc: true
mermaid: false
fancybox: false
blueprint: false
# latex support
katex: true
markup: mmark
mmarktoc: true
# UEVersion: 5.3.2 
---


有一个很有意思的文件，bsdftest.cpp

其实这就是我一直想实现的白炉测试了。 (uniform incoming radiance: 1.0)


# BRDF能量守恒


$$
 \int_{\Omega} f(\omega_i, \omega_o) \cos(\omega_i,n) d\omega_i <= 1.0
$$

# Lambert BRDF


当$$\rho = 1.0$$时候，这个BRDF是完全不会丢失能量的。



# 强弱白炉测试

原始论文见: https://www.jcgt.org/published/0003/02/03/paper.pdf

C++实现见: https://github.com/knarkowicz/FurnaceTest/blob/master/main.cpp



要看懂这个文件，首先要看懂 几何遮蔽函数。
我们在实时渲染里写的那个只是个简化的方案。

PBRT给了一个比较正常的公式
https://www.pbr-book.org/3ed-2018/Reflection_Models/Microfacet_Models#MaskingandShadowing

![edit-2ee8470b2d47485b9a0149cf8e26c7f1-2024-08-03-19-31-05](https://img.blurredcode.com/img/edit-2ee8470b2d47485b9a0149cf8e26c7f1-2024-08-03-19-31-05.png?x-oss-process=style/compress)


另外白炉测试的时候:

- 假设光照来自各个方向的光照是均匀的 1$$sr$$
- 如果有菲涅尔，假设菲涅尔的值为恒定的1.0
 
https://github.com/QianMo/PBR-White-Paper/blob/master/content/part%205/README.md#52-%E5%BC%B1%E7%99%BD%E7%82%89%E6%B5%8B%E8%AF%95the-weak-white-furnace-test

## 强白炉测试


$$
\int_{\Omega} \frac{G(\omega_i, \omega_o,n) D(n,h,a)}{4 |n \cdot \omega_o|}  d\omega_i = 1.0
$$

这里经过重要性采样，D项可以被彻底消掉

推导过程可以见[法线分布函数ndf](
https://www.blurredcode.com/2021/05/dec701b2/#%e6%b3%95%e7%ba%bf%e5%88%86%e5%b8%83%e5%87%bd%e6%95%b0ndf)


## 弱白炉测试

几何遮蔽可以分为两部分 : 光源方向进来的遮蔽，和经过反弹以后的光线被遮蔽，无法到达观察方向。

弱白炉测试就是不考虑光源方向进来的遮蔽，只考虑反射后的遮蔽。


弱白炉测试的公式

$$
\int_{\Omega} \frac{G_1(\omega_o, n) D(n,h,a)}{4 |n \cdot \omega_o|}  d\omega_i = 1.0
$$



对比这两个函数的区别

```cpp
float WeakWhiteFurnaceTest(float roughness, float ndotv)
{
    ...
	for (unsigned i = 0; i < sampleNum; ++i)
	{
        ...
		float const g1 = SmithG1(roughness, ndotv);  // 注意这一行,若白炉测试不考虑光源方向的遮蔽
		float const pdf = 4.0f * vdoth / ndoth; 
		integral += (g1 * pdf) / (4.0f * ndotv);
	}
	integral /= sampleNum;
	return Saturate(integral);
}

float WhiteFurnaceTest(float roughness, float ndotv)
{
    ...
	for (unsigned i = 0; i < sampleNum; ++i)
    {
        ...
		float const g = SmithG(roughness, ndotv, ndotl);  // 以及这一行
		float const pdf = 4.0f * vdoth / ndoth;
		integral += (g * pdf) / (4.0f * ndotv);
    }
	integral /= sampleNum;
	return Saturate(integral);
}
```

# PBRT3的BRDF测试


## 弱白炉测试
位置在`bsdftest.cpp`里，测试光源在正上方直射的情况下，BRDF的积分值。
更接近于弱白炉测试。
因为ndotv始终等于1.


```cpp
        // facing directly at normal
        Vector3f woL = Normalize(Vector3f(0, 0, 1));
        Vector3f wo = bsdf->LocalToWorld(woL);
        // was bsdf->dgShading.nn
        const Normal3f n = Normal3f(bsdf->LocalToWorld(Vector3f(0, 0, 1)));
        // for each method of generating samples over the hemisphere
        for (int gen = 0; gen < numGenerators; gen++) {
			...
```

## 重要性采样测试

对
$$ \int_{\Omega^+} 1 d{w}$$
 这个积分进行了验证，拆成MC形式应该是

$$ \sum_{0}^{N} \frac{1}{pdf(w_i)} * \frac{1}{N}$$

这个式子的结果应该是 2Pi。

更准确的说，给定一个BRDF 的一个方向 `wo`，对 BRDF 重要性采样生成的方向 `wi`，其半球积分值应该接近于2Pi，否则说明重要性采样写的有问题。


```cpp
Vector3f woL = Normalize(Vector3f(0, 0, 1));
Vector3f wo = bsdf->LocalToWorld(woL);
// was bsdf->dgShading.nn
const Normal3f n = Normal3f(bsdf->LocalToWorld(Vector3f(0, 0, 1)));
	for (int sample = 0; sample < estimates; sample++) {
		Vector3f wi;
		Float pdf;
		Spectrum f;
		// sample hemisphere around bsdf, wo is fixed
		(SampleFuncArray[gen])(bsdf, wo, &wi, &pdf, &f);
		// 统计半球积分值
			histogram[histoCosTheta][histoPhi] += 1.0 / pdf;
	}
	...
```

可以看 对Lambert BRDF 做的测试结果
```cpp
*** BRDF: 'Lambertian', Samples: 'Cos Hemisphere'

wi histogram showing the relative weight in each bin
  all entries should be close to 2pi = 6.28319:
  (0 bad samples, 0 outside samples)

                          phi bins
  cos(theta) bin 00:  6.41  6.16  6.27  6.18  6.27  6.29  6.32  5.95  6.13  6.24
  cos(theta) bin 01:  6.25  6.21  6.33  6.24  6.31  6.32  6.27  6.31  6.27  6.32
  cos(theta) bin 02:  6.26  6.28  6.32  6.27  6.32  6.27  6.22  6.30  6.27  6.22
  cos(theta) bin 03:  6.31  6.30  6.31  6.31  6.27  6.31  6.28  6.31  6.29  6.26
  cos(theta) bin 04:  6.32  6.25  6.31  6.30  6.24  6.31  6.25  6.25  6.30  6.26
  cos(theta) bin 05:  6.28  6.29  6.26  6.28  6.27  6.27  6.28  6.26  6.28  6.29
  cos(theta) bin 06:  6.30  6.26  6.27  6.28  6.26  6.28  6.29  6.28  6.29  6.27
  cos(theta) bin 07:  6.27  6.28  6.28  6.27  6.28  6.30  6.28  6.27  6.30  6.28
  cos(theta) bin 08:  6.27  6.30  6.30  6.28  6.28  6.31  6.27  6.28  6.29  6.29
  cos(theta) bin 09:  6.26  6.29  6.30  6.31  6.27  6.28  6.28  6.30  6.29  6.32

  final average :  6.27657 (error -0.00661)

  radiance = 1.00000
```

- 每个bin的值都接近于2Pi,说明重要性采样的 pdf 没啥问题
- radiance = 1.0，说明没有能量损失


## 卡方检验

Nori 和 PBRT 里其实都有这个的实现，关键词`ChisquareTest` / `Chi2Test`，但是我概率论已经全还给老师了，看不懂了。