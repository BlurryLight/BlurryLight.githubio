
---
title: "UE4 Hide Stats Rendering"
date: 2022-11-28T23:09:05+08:00
draft: false
categories: [ "UE"]
isCJKLanguage: true
slug: "3408630c"
toc: false
mermaid: false
fancybox: false
# latex support
# katex: true
# markup: mmark
# mmarktoc: false 
UEVersion: 4.26.2
---

{{< zhTranslation "UE4 隐藏统计数据Stats渲染" >}} 

Recently there was a need for hide UE4 `stats` rendering.

When stats are enabled, the stats data are collected and meanwhile tables are rendering into viewport canvas.
The need is to collect stats data background without rendering them on the canvas, which obscures other primitives.
Using `stats none` or other similar methods like [[Solved] How to disable fps counter in rendered level sequence? - Development / Rendering - Unreal Engine Forums](https://forums.unrealengine.com/t/solved-how-to-disable-fps-counter-in-rendered-level-sequence/444255/4) is not possible because it also stops statistics collection in background silently.

# Two kinds of stats

There are actually two kinds of `stats` in UE4 Engine, as shown below.

![UE4HideStatsRendering-2022-11-28-23-15-23](https://img.blurredcode.com/img/UE4HideStatsRendering-2022-11-28-23-15-23.png?x-oss-process=style/compress)


## EngineStats

The Type1 stats(EngineStats) function pointers are actually stored in `Engine.h`:

```cpp
TArray<FEngineStatFuncs> EngineStats;
```

```cpp
...
EngineStats.Add(FEngineStatFuncs(TEXT("STAT_NamedEvents"), TEXT("STATCAT_Engine"), FText::GetEmpty(), &UEngine::RenderStatNamedEvents, &UEngine::ToggleStatNamedEvents, bIsRHS));
EngineStats.Add(FEngineStatFuncs(TEXT("STAT_FPS"), TEXT("STATCAT_Engine"), FText::GetEmpty(), &UEngine::RenderStatFPS, &UEngine::ToggleStatFPS, bIsRHS));
EngineStats.Add(FEngineStatFuncs(TEXT("STAT_Summary"), TEXT("STATCAT_Engine"), FText::GetEmpty(), &UEngine::RenderStatSummary, NULL, bIsRHS));
EngineStats.Add(FEngineStatFuncs(TEXT("STAT_Unit"), TEXT("STATCAT_Engine"), FText::GetEmpty(), &UEngine::RenderStatUnit, &UEngine::ToggleStatUnit, bIsRHS));
...
```

These engine stats are somewhat hard to hide.
We must hack into **every** `UEngine::RenderStatxxx` to disable the logic about rendering elements on canvas if we want to hide all Engine Stats.
That's because the logic about collect statistics and rendering them is mixed. 
Skipping the `RenderStatXXX` function call, we will get wrong statistics because calculations are also skipped.

Take `Stat Unit` as example.


```cpp
int32 FStatUnitData::DrawStat(FViewport* InViewport, FCanvas* InCanvas, int32 InX, int32 InY)
{
    // some logic to calculate FrameTime/RenderThreadTime/...
    ....

    RawFrameTime = DiffTime * 1000.0f;
    FrameTime = 0.9 * FrameTime + 0.1 * RawFrameTime;
    ...
    RawGameThreadTime = FPlatformTime::ToMilliseconds(GGameThreadTime);
    GameThreadTime = 0.9 * GameThreadTime + 0.1 * RawGameThreadTime;

    /** Number of milliseconds the renderthread was used last frame. */
    RawRenderThreadTime = FPlatformTime::ToMilliseconds(GRenderThreadTime);
    RenderThreadTime = 0.9 * RenderThreadTime + 0.1 * RawRenderThreadTime;

    RawRHITTime = FPlatformTime::ToMilliseconds(GRHIThreadTime);
    RHITTime = 0.9 * RHITTime + 0.1 * RawRHITTime;

    RawInputLatencyTime = FPlatformTime::ToMilliseconds64(GInputLatencyTime);
    InputLatencyTime = 0.9 * InputLatencyTime + 0.1 * RawInputLatencyTime;

    ....
    SET_FLOAT_STAT(STAT_UnitFrame, FrameTime);
    SET_FLOAT_STAT(STAT_UnitRender, RenderThreadTime);
    SET_FLOAT_STAT(STAT_UnitRHIT, RHITTime);
    SET_FLOAT_STAT(STAT_UnitGame, GameThreadTime);
    SET_FLOAT_STAT(STAT_UnitGPU, GPUFrameTime[0]);
    SET_FLOAT_STAT(STAT_InputLatencyTime, InputLatencyTime);

    //inject our logic here to early return
    static auto CVar = IConsoleManager::Get().FindConsoleVariable(TEXT("r.StatsRendering"));
    if (!CVar->GetBool())
    {
        return InY;
    }

    ...
    // Draw Them on Canvas
    // Draw unit.
    {
        int32 X3 = InX * (bStereoRendering ? 0.5f : 1.0f);
        if (bShowUnitMaxTimes)
        {
            X3 -= (int32)((float)Font->GetStringSize(TEXT(" 000.00 ms ")));
        }

        int32 X2 = bShowUnitMaxTimes ? X3 - (int32)((float)Font->GetStringSize(TEXT(" 000.00 ms "))) : X3;
        int32 X1 = X2 - (int32)((float)Font->GetStringSize(TEXT("DynRes: ")));
        const int32 RowHeight = FMath::TruncToInt(Font->GetMaxCharHeight() * 1.1f);

```

We need to inject following code in the middle of the function, after calculation FrameTime/GameThreadTime/... and other, to early return, skipping canvas rendering code.

```cpp
static auto CVar = IConsoleManager::Get().FindConsoleVariable(TEXT("r.StatsRendering"));
if (!CVar->GetBool())
{
    return InY;
}
```


## Group Stats

Type2 stats have same rendering entry point in `StatsRender2.cpp` as

```cpp
void RenderStats(FViewport* Viewport, class FCanvas* Canvas, int32 X, int32 Y, int32 SafeSizeX)
```
### Way1 GRenderStats

There is a global variable `bool GRenderStats` in `StatsCommmand.cpp` to control the stats rendering.
The variable can be populated by command `-nodisplay `.

Example:

```
stat rhi -nodisplay
```

The RHI group stats will be collected, but `GRenderStats` will be false.
So tables are not drawn on canvas.


### Way2 user-defined cvar

We can also take the approach we used before to hide tables. 
The mind behind it is to early return from `RenderStats` before the actual rendering funcs are called.
We insert our code in `RenderStats()`.

```cpp
void RenderStats(FViewport* Viewport, class FCanvas* Canvas, int32 X, int32 Y, int32 SafeSizeX)
{
    DECLARE_SCOPE_CYCLE_COUNTER( TEXT( "RenderStats" ), STAT_RenderStats, STATGROUP_StatSystem );

    FGameThreadStatsData* ViewData = FLatestGameThreadStatsData::Get().Latest;

    //++
    static auto CVar = IConsoleManager::Get().FindConsoleVariable(TEXT("r.StatsRendering"));
    if (!ViewData || !ViewData->bRenderStats || !Cvar->GetBool())
    {
        return;
    }
    //--
    
    FStatRenderGlobals& Globals = GetStatRenderGlobals();
    // SizeX is used to clip/arrange the rendered stats to avoid overlay in stereo mode.
    const bool bIsStereo = Canvas->IsStereoRendering();
    Globals.Initialize( Viewport->GetSizeXY().X/Canvas->GetDPIScale(), Viewport->GetSizeXY().Y/Canvas->GetDPIScale(), SafeSizeX, bIsStereo );


    if( !ViewData->bDrawOnlyRawStats )
    {
        RenderGroupedWithHierarchy(*ViewData, Viewport, Canvas, X, Y);
    }
    ....
```




