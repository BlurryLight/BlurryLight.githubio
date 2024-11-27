
---
title: "Rider在虚幻的调试技巧"
date: 2024-11-27T02:32:17Z
draft: false 
categories: [ "UE"]
isCJKLanguage: true
slug: "4bdaa5f3"
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

# 查看全局变量

![edit-c34d26b7f74a4540b6e52f77a95fdbb1-2024-01-27-15-48-25](https://img.blurredcode.com/img/edit-c34d26b7f74a4540b6e52f77a95fdbb1-2024-01-27-15-48-25.png?x-oss-process=style/compress)

VS的语法


```
UnrealEditor-Core!GConfig
UnrealEditor-Engine!GPlayInEditorContextString
```


Rider的语法:

```cpp
{,,UnrealEditor-UnrealEd.dll}::GEditor->PreviewPlatform.PreviewFeatureLevel
{,,UnrealEditor-RHI.dll}::GMaxRHIFeatureLevel
{,,UnrealEditor-UnrealEd.dll}::GEditor->PreviewPlatform.PreviewFeatureLevel
{,,UnrealEditor-Core}::PrintScriptCallstack() // 查看蓝图堆栈
{,,UnrealEditor-Engine.dll}::GPlayInEditorContextString
```

# 条件断点


`FString`:  ` wcsstr((wchar_t*)MyString.Data.AllocatorInstance.Data, L"Search substring") `

`FName`:  `strstr(((FNameEntry&)GNameBlocksDebug[MyFName.DisplayIndex.Value >> FNameDebugVisualizer::OffsetBits][FNameDebugVisualizer::EntryStride * (MyFName.DisplayIndex.Value & FNameDebugVisualizer::OffsetMask)]).AnsiName,"Searchsubstring")`


# Reference:

更详细的资料可以从这里找到，这片笔记的内容都包含在这片好文里
https://dev.epicgames.com/community/learning/tutorials/dXl5/advanced-debugging-in-unreal-engine