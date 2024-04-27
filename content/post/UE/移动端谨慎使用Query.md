
---
title: "UE | 移动端谨慎使用Occlusion/Timer Query"
date: 2024-04-27T15:29:06+08:00
draft: false
categories: [ "UE"]
isCJKLanguage: true
slug: "42bea8b2"
toc: true 
mermaid: false
fancybox: false
blueprint: false
# latex support
# katex: true
# markup: mmark
# mmarktoc: false 
---

最近在项目里连续遇到了几个关于Query的问题，这里记录一下..

# Occlusion Query

Hardware Occlusion Query在桌面端是一个很可靠的功能， GPU Gems里有好几篇文章都介绍过这个功能用来做开销，也是从零几年就被广泛使用的老技术了，稳定可靠。

> 参考：[Chapter 6. Hardware Occlusion Queries Made Useful | NVIDIA Developer](https://developer.nvidia.com/gpugems/gpugems2/part-i-geometric-complexity/chapter-6-hardware-occlusion-queries-made-useful)


但是在移动端还是发现他比较不可靠，主要问题集中在:

1. 发起一次Occlusion Query需要至少3次API Call，Query多的时候可能会导致性能问题
2. 部分API和硬件对Occlusion Query的数量有隐式限制。

先谈第一个问题，以vk作为例子，发起一次Occlusion Query至少需要以下API Call:

```cpp
vkCmdBeginQuery(commandBuffer, queryPool, 0, 0);

// draw something

// End occlusion query
vkCmdEndQuery(commandBuffer, queryPool, 0);
uint64_t queryResult;
vkGetQueryPoolResults(
```

在gl上也是对应的，在GL上会有一定的开销。这个问题在tile-based renderer上可能更明显，因为bin的数量可能会放大Query执行的次数。


第二个问题则更加隐蔽，因为无论是OpenGL / Vulkan的Spec都没有规定Query的数量上限。

但是根据 webgpu上的一个issue的讨论， 在Metal上对Occlusion Query应该是有一个隐蔽的buffer大小限制，最多只能有8192个。

> 参考：[Query API: Maximum limit on query count · Issue #1324 · gpuweb/gpuweb](https://github.com/gpuweb/gpuweb/issues/1324)


另外一个很隐蔽的限制，就是Adreno的gpu文档上专门有一条写着，

> *How many Occlusion Queries should I use?*
> 
> *No more than **512** should be active. There is usually a three (3) frame delay for results.*


尽管在真机上测试的时候，创建了超过512个Query也不会报错，但是我在vivo的某台机器上法线，当场景很复杂且Query数量很多的情况下(几千个)，可能会导致vkGetQueryPoolResults超时。(虚幻默认0.5s超时)

此外，在虚幻的代码里，如果是GLes的情况，也对Query的数量有一定限制。

在`Engine/Source/Runtime/OpenGLDrv/Private/Android/AndroidOpenGL.cpp`里有一个最大限制

```
static int32 GMaxmimumOcclusionQueries = 4000;
```


此外，如果是Adreno平台，这个上限还会更小，只有510的Query可以用。

![移动端谨慎使用Query-2024-04-27-15-47-41](https://img.blurredcode.com/img/移动端谨慎使用Query-2024-04-27-15-47-41.png?x-oss-process=style/compress)



# Timer Query

Timer Query的主要问题是在移动端使用它意义不大，测出来的数据可能会有很大的误差。
Unreal Insights的GPU 耗时也是通过Timer Query来测量的，所以在移动端上抓出来的数据经常让人啼笑皆非，比如测量出来BasePass竟然时间还没有PostProcess长。


这里引用一下Arm工程师在Vulkaised 2023会议上的一个演讲(https://www.youtube.com/watch?v=BD1zXW7Uz8Q),

![移动端谨慎使用Query-2024-04-27-15-56-57](https://img.blurredcode.com/img/移动端谨慎使用Query-2024-04-27-15-56-57.png?x-oss-process=style/compress)

## 测量RenderPass

在没有依赖关系之间的RenderPass的VS/PS可以任意调度，有可能在某个RenderPass的VS/PS中间插入了其他RenderPass的VS/PS，这样就会导致Timer Query测量出来的时间被意外延长。


## 测量DrawCall

这个问题更大，因为tiled based的结果，PS可能被分解到很多个tile执行，时间和其他的drawcall混在一起了，测出来是个没意义的数据。

