
---
title: "Unreal Including IndirectDraw Primitives In Stats(DirectX12 Only)"
date: 2024-06-17T23:12:00+08:00
draft: false
categories: [ "UE"]
isCJKLanguage: true
slug: "aeefcfc1"
toc: false
mermaid: false
fancybox: false
blueprint: false
# latex support
# katex: true
# markup: mmark
# mmarktoc: false 
---


# Stat Unit / Stat RHI

`IndirectDraw` is widely used in Unreal (especially on Desktop Deferred Renderer).
It's a common practice to use `IndirectDraw` to draw a large number of objects with a single draw call, with GPU Culling on GPUScene.
With GPUScene enabled, `HISM` and `ISM` and `landscape` will be drawn with `IndirectDraw`.

However, the instances drawn with `IndirectDraw` are decided by Compute Shader, the result is stored in a GPU Buffer and then used in the `IndirectDraw` call. There is no simple way to detect how much primitives are drawn.

So, `Stat Unit` and `Stat RHI` simply ignore those primitives.
In large open world with full of Foliages and Grass, there may be a huge gap between them and the actual primitives are processed on GPU.


For example:

I made a simple scene with about 10 trees with FoliageActor(actually `HISM` ), and the `stat rhi` reported drawing about 4000 triangles.


![Unreal_Including_IndirectDraw_In_Statistics-2024-06-17-23-23-49](https://img.blurredcode.com/img/Unreal_Including_IndirectDraw_In_Statistics-2024-06-17-23-23-49.png?x-oss-process=style/compress)


Actually there are about 400'000 triangles drawn, reported by Renderdoc.

![Unreal_Including_IndirectDraw_In_Statistics-2024-06-17-23-28-32](https://img.blurredcode.com/img/Unreal_Including_IndirectDraw_In_Statistics-2024-06-17-23-28-32.png?x-oss-process=style/compress)


# Pipeline Statistics Query

How to get the actual number of primitives drawn with `IndirectDraw`?

The first thing comes to mind is reading back the GPU Buffer and counting the number of instances drawn.
However this is not a good idea as it can be extremely slow and may cause a stall.

Another approach is to use GPU Profiling/Debugging Tools such as RenderDoc / XCode and Snapdragon Profiler. These tools collect information from the Hardware Counter or the Hooked Graphics API ,providing accurate numbers. One of the advantages of this approach is that it is graphics API independent and transparent, eliminating the need to modify the code.

The third way is to use `Pipeline Statistics Query` in the Graphics API.

- For OpenGL, `Primitive queries` with `GL_PRIMITIVES_GENERATED​​` are needed to get the number of primitives output by the `Vertex Processing Step`.
- For Vulkan, `VK_QUERY_PIPELINE_STATISTIC_INPUT_ASSEMBLY_VERTICES_BIT` can be used to get the number of vertices processed by the Input Assembly.
- For D3d12, `D3D12_QUERY_DATA_PIPELINE_STATISTICS` gives number of primitives read by the input assembler. 

In Unreal5, only the `D3d12` backend has some basic support for `Pipeline Statistics Query`.

The command **stat d3d12rhipipeline** provides pipeline statistics.

![Unreal_Including_IndirectDraw_In_Statistics-2024-06-17-23-37-23](https://img.blurredcode.com/img/Unreal_Including_IndirectDraw_In_Statistics-2024-06-17-23-37-23.png?x-oss-process=style/compress)