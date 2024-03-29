
---
title: "UE Cull Distance Volume 杂记"
date: 2023-12-03T23:41:57+08:00
draft: false
categories: [ "UE"]
isCJKLanguage: true
slug: "547b212a"
toc: true
mermaid: false
fancybox: false
blueprint: false
# latex support
# katex: true
# markup: mmark
# mmarktoc: false 
---



UE的距离剔除不是默认打开的，需要在场景里摆一个`Cull Distance Volume`打开才生效。
> 参考：[Cull Distance Volume | Unreal Engine 4.27 Documentation](https://docs.unrealengine.com/4.27/en-US/RenderingAndGraphics/VisibilityCulling/CullDistanceVolume/)


官方文档写的还算详细，补充几点杂记。

# 在编辑器下预览Cull Distance Volume

需要按G进入`GameView`，隐藏所有的gizmo或者要进入PIE的情况下才能正确预览cull distance生效。


# 生效的条件

- Actor是Static
- Actor需要勾选bAllowCullDistanceVolume
- 没有勾选HiddenInGame，bVisible也是true

![edit-76e7e1a6ec27450499a5bdbe6e981af5-2023-11-27-22-51-29](https://img.blurredcode.com/img/edit-76e7e1a6ec27450499a5bdbe6e981af5-2023-11-27-22-51-29.png?x-oss-process=style/compress)

# 具体应用的距离

tldr: 以Bounds的直径为准，找到abs差最接近的Size到CullDistance的映射。

详细过程:

`Volume`里保存了一堆`FCullDistanceSizePair`结构体， 分别表示`Size -> CullDistance`的映射

核心循环体是
- 获得PrimitiveComponent的BoundsRadius, 乘以2获得直径
- 遍历`FCullDistanceSizePair`数组，找到最接近的Size (**abs差最小**)
- 以记录的CullDistance作为PrimitiveComponent的CullDistance

```cpp
if (EncompassesPoint(PrimitiveComponent->GetComponentLocation()))
{		
    // Find best match in CullDistances array.
    const float PrimitiveSize = PrimitiveComponent->Bounds.SphereRadius * 2;
    float CurrentError = FLT_MAX;
    float CurrentCullDistance = 0.f;
    for (const FCullDistanceSizePair& CullDistancePair : CullDistances)
    {
        const float Error = FMath::Abs( PrimitiveSize - CullDistancePair.Size );
        if (Error < CurrentError)
        {
            CurrentError = Error;
            CurrentCullDistance = CullDistancePair.CullDistance;
        }
    }

    float& CullDistance = PrimitiveMaxDistancePair.Value;

    // LD or other volume specified cull distance, use minimum of current and one used for this volume.
    if (CullDistance > 0.f)
    {
        CullDistance = FMath::Min(CullDistance, CurrentCullDistance);
    }
    // LD didn't specify cull distance, use current setting directly.
    else
    {
        CullDistance = CurrentCullDistance;
    }
}
```

# Cull Distance Volume应用的时机

实质上是会调用`UWorld::UpdateCullDistanceVolumes(AActor* ActorToUpdate = nullptr, UPrimitiveComponent* ComponentToUpdate = nullptr);`
对Volume的编辑会导致World的下次tick更新所有的Actor，通过 `GetWorld()->bDoDelayedUpdateCullDistanceVolumes = true;`


此外在World加载，或者Actor/Component的加载时，也会调用这个API来更新自己的CullDistance。

最后会通过 `MarkRenderStateDirty`，重新创建SceneProxy。

![edit-76e7e1a6ec27450499a5bdbe6e981af5-2023-11-27-23-15-10](https://img.blurredcode.com/img/edit-76e7e1a6ec27450499a5bdbe6e981af5-2023-11-27-23-15-10.png?x-oss-process=style/compress)


![edit-76e7e1a6ec27450499a5bdbe6e981af5-2023-11-27-23-16-59](https://img.blurredcode.com/img/edit-76e7e1a6ec27450499a5bdbe6e981af5-2023-11-27-23-16-59.png?x-oss-process=style/compress)


## 剔除发生的时机


- 正常渲染的流程中，剔除发生在`InitViews`阶段的`FrustumCull`函数中，随着视椎体的剔除一起剔了。

```cpp
    bool bDistanceCulled = DistanceSquared > FMath::Square(MaxDrawDistance + FadeRadius) || (DistanceSquared < MinDrawDistanceSq);

    // Store distane culled primitives so it can correctly culled when collecting RT primitives
    if (bDistanceCulled)
    {
        DistanceCulledBits |= Mask;
    }
    ...
```

- ShadowSetup过程中发生在 `bool FViewInfo::IsDistanceCulled_AnyThread`

- HLOD的渲染中，`void FLODSceneTree::UpdateVisibilityStates(FViewInfo& View)`也会检查DistanceCull。
