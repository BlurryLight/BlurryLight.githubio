
---
title: "Talking about Hitches in UE4 LoadingScreen"
date: 2022-12-10T23:45:16+08:00
draft: false
categories: [ "UE"]
isCJKLanguage: true
slug: "6ec75576"
toc: true 
mermaid: false
fancybox: false
# latex support
# katex: true
# markup: mmark
# mmarktoc: false 
UEVersion: 4.26.2
---

{{< zhTranslation "谈谈UE4 LoadingScreen卡顿问题" >}} 

This blog is just some casual thoughts around one problem I've met.
I believe there are ways to solve it, However I've yet found.

# Background

Loading Screen is an common component of games, preventing users from getting bored by wait game loading.
Unreal provides `MoviePlayer` module for the task. 
It allows for playing an `mp4` movie and presenting an animated slate(UMG is actually supported, but needs to modifier engine to suppress an `ensure` in 4.26, see [Appendix: Support UMG in LoadingScreen](#a-support-umg-in-loadingscreen)).

There are several tutorials or libraries about this functionality, such as following:
- [Loading Screen | Unreal Engine Community Wiki](https://unrealcommunity.wiki/loading-screen-243mzpq1)
- [truong-bui/AsyncLoadingScreen](https://github.com/truong-bui/AsyncLoadingScreen)

The underlying implementation of those solutions is consistent, which is `MoviePlayer`.
Behavior may vary for different configurations, such as *Should the movie ends playing automaticlly when map loading ends?*
However, the basic logic is same:

1. `GameThread` is Loading Map
2. When Movie is present, `RenderThread` is responsible for decoding/presenting it
3. A transient thread named `SlateLoadingThread` ticks slate animations.
4. when `GameThread` completes map loading, it loops in `WaitForMovieFinish` waiting for movie playing, terminates `SlateLoadingThread` and takes over its task, and begins to tick

The `MoviePlayer` module is much more complicated than what is describled above, and a great deal of details, such as synchronization between 3 threads, are in code.

## Problem 

I've found when LoadingScreen is playing an mp4 movie, there would be noticable lags/hitchs at some fixed timepoints. 
After days of investigation with `Unreal Insights`, I found the suspects.

As describled above, `GameThread` is loading map(including persistent level and streaming levels), which consists of many assets.
`RenderingThread` is ticking at fixed rate(60HZ) for playing mp4.
When `GameThread` completes **ONE** umap loading, there is an `FlushAsyncLoading` command, and all assets in umap will create their GPU Resource:
StaticMeshs will create and upload`VertexBuffer` and `IndexBuffer`, images will create Textures and upload image data and so on.

We take `UTexture::PostInit` as example, some unrelated code is removed (noted by `//...`)for simplicity:

```cpp
void UTexture::UpdateResource()
{
    // Release the existing texture resource.
    ReleaseResource();

    //Dedicated servers have no texture internals
    if( FApp::CanEverRender() && !HasAnyFlags(RF_ClassDefaultObject) )
    {
        // Create a new texture resource.
        Resource = CreateResource();

        if (Resource)
        {
            // .. some other code

            // Init the texture reference, which needs to be set from a render command, since TextureReference.TextureReferenceRHI is gamethread coherent.
            FTextureResource* ResourceToInit = Resource;
            ENQUEUE_RENDER_COMMAND(SetTextureReference)([this, ResourceToInit](FRHICommandListImmediate& RHICmdList)
            {
                ResourceToInit->SetTextureReference(TextureReference.TextureReferenceRHI);
            });
            BeginInitResource(Resource);

            // ...
        }
    }
}

void BeginInitResource(FRenderResource* Resource)
{
    ENQUEUE_RENDER_COMMAND(InitCommand)(
        [Resource](FRHICommandListImmediate& RHICmdList)
        {
            Resource->InitResource();
        });
}

void FRenderResource::InitResource()
{
    check(IsInRenderingThread());
    
    // ... some other code
    InitDynamicRHI();
    InitRHI();

    // ... some other code
}
```

which steps to here:

```cpp
void FTexture2DResource::CreateTexture()
{
    // ...
    Texture2DRHI = RHICreateTexture2D( RequestedMip->SizeX, RequestedMip->SizeY, PixelFormat, State.NumRequestedLODs, 1, CreationFlags, CreateInfo);
    // ...

    // ...
    // Read the resident mip-levels into the RHI texture.
    for (int32 RHIMipIdx = 0; RHIMipIdx < State.NumRequestedLODs; ++RHIMipIdx)
    {
        const int32 ResourceMipIdx = RHIMipIdx + RequestedMipIdx;
        if (MipData[ResourceMipIdx])
        {`
            uint32 DestPitch;
            void* TheMipData = RHILockTexture2D( Texture2DRHI, RHIMipIdx, RLM_WriteOnly, DestPitch, false );
            // Memcpy inside
            GetData( ResourceMipIdx, TheMipData, DestPitch );
            RHIUnlockTexture2D( Texture2DRHI, RHIMipIdx, false );
        }
    }
    // ...
}
```

In `UTexture::PostLoad`, we enqueue tasks to `RenderThread`, and one of the tasks is to allocate GPU buffer for the texture, 
and then uploading image data(including mipmaps) to the texture object. 
This happens on `RenderThread`.

Remember, our movie is currently also playing on `RenderThread`.
When there are thousands of textures uploading(easy to acheive when loading a medium size of `umap`), 
the `Renderthread` gets stucking at uploding and therefore cannot spare time to tick the video, that's the lag/hitch.

## Asychronous Texture Uploading

Asychronous Texture Uploading, as far as I know, exists at least in DirectX 11 and OpenGL. 
I'm not familiar with other APIs.

Unreal also have its own implementation:

```cpp
TextureRHI = RHIAsyncCreateTexture2D( RequestedMip->SizeX, RequestedMip->SizeY, PixelFormat, 1, TexCreate_ShaderResource,MipData.GetData() ,1);
```

In Unreal 4.26.2, Only `OpenGLRHI` and `DirectX11` RHI implements the function.
`Vulkan/DirectX12` have an empty implementation and will abort when called.
I've looked through the code in UE5 and this feature is also implemented in DX12/Vulkan in UE5.

![UE4_Asyncchronous_texture_creation_uploading-2022-12-11-00-29-09](https://img.blurredcode.com/img/UE4_Asyncchronous_texture_creation_uploading-2022-12-11-00-29-09.png?x-oss-process=style/compress)

There is an excellect plugin and essay introduced the function and its usage, so I'm not going to talk much about it.
> 参考：[Peter Leontev - Entrepreneur & Game Tools & Tech Programmer | Efficient and asynchronous creation of textures at runtime in Unreal Engine](https://peterleontev.com/blog/efficient_construction_of_textures/)

However, the problem has yet solved. There are 3 ways I've thought to sovle it,

1. To change the underlying implementation of `UTexture::PostLoad` from synchonours to asynchronous, However I'm afraid this may cause serious synchronous problems. 
It may requires much tests to ensure the correctness and robustness, and the silent change in such an low-level implementation bring some trouble for other programmers.
2. To Move the `CreateTexture` and `Lock/Memcpy/Unlock` from `RenderThread` to other Thread, such as `RHIThread`. However it may also cause some synchronous problems.
3. Rewrite the `MoviePlayer` to make the movie plays on other thread and make the `RenderThread` dedicated to preparing shading resources.

All of those solutions are too complicated for me to complete alone in days.
Therefore, the problem remains unresovled.


# Appendix

## a. Support UMG in LoadingScreen

**This behavior/solution maybe only valid in UE 4.26**.

In `Engine/Source/Runtime/UMG/Private/UserWidget.cpp`, there is

```cpp
void UUserWidget::NativeTick(const FGeometry& MyGeometry, float InDeltaTime)
{ 
    // ...
        if (bTickAnimations)
        {
            // warning: it will cause an ensure error
            if (!CVarUserWidgetUseParallelAnimation.GetValueOnGameThread())
            {
    //...
```

As we have just described, when loading screen is working, the slate ticks on a transient thread named `SlateLoadingThread`.
Therefore, if an `UMG` is employed as LoadingScreen UI, it will throw an `ensure` error in this line:

```cpp
CVarUserWidgetUseParallelAnimation.GetValueOnGameThread()
```

Just modify it to

```cpp
CVarUserWidgetUseParallelAnimation.GetValueOnAnyThread()
```

When `SlateLoadingThread` is alive, `GameThread` is busying loading umaps and `RenderThread` is ticking movie if avilable, or waiting for tasks such as preparing shading resources.
Therefore there are no other threads who will try to the modify the UMG. 
In this case, it's still thread-safe.

## b. OpenGL Async Buffer Uploading/Downloading With Pixel Buffer Object

There are two excellent materials about the topic:
- [OpenGL Pixel Buffer Object (PBO)](http://www.songho.ca/opengl/gl_pbo.html)
- OpenGL Insights Chapter 28 "Asynchronous Buffer Transfers"

For uploading with PBO, `glTexImage2D` will immediately return, and the uploading process begins background and silently. 
OpenGL Driver employs implicit fence to make sure when `glDrawXXX` emits, the async resources are ready.

For downloading with PBO, `glReadPixels` will immediately return. 
When `glMapBufferRange` emits, OpenGL driver will make sure the downloading is completed.


I've implemented async Texture Uploading/Downloading in [PboBench.cc](https://github.com/BlurryLight/DiRenderLab/blob/main/examples/PboBench/PboBench.ccthe), in style named `Unsynchronized Buffers` by <<OpenGL Insights>> chapter 28.
In this style, we use `glFenceSync` and `GL_MAP_UNSYNCHRONIZED_BIT` to disable implicit sync in OpenGL driver and control sync by ourselves, which gives full flexibility and performance advantage.
