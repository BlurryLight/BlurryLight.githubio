
---
title: "Quick Reference About Asset Name/Path in UE"
date: 2023-05-20T11:41:47+08:00
draft: false
categories: [ "UE"]
isCJKLanguage: true
slug: "7a83b54b"
toc: true
mermaid: false
fancybox: false
# latex support
# katex: true
# markup: mmark
# mmarktoc: false 
UEVersion: 4.26.2
---


There are tons of different functions that return `AssetName`/`AssetPath`/`PackagePath`/`PackageName`, but there are subtle differences in the strings returned by different APIs.
What's worse, there are also many different APIs for manipulating assets that accept randomly formatted path strings. Some of them accept `Virtual Path` defined by UE, while others may accept `System Path`.

I'm tired of using `Go to Definition` to check what format an API needs. So I made a table to record what an API will return/need.

## UObject Related

| UObject Related      |                                                 | note                                         |
| :-------------------- | ----------------------------------------------- | -------------------------------------------- |
| original             | `/Game/Meshes/Test/Material.Material`           |                                              |
| GetFName()/GetName() | UObject return `Material`                       | UPackage return `/Game/Meshes/Test/Material` |
| **GetPathName()**    | `'/Game/Meshes/Test/Material.Material'`         |                                              |
| GetFullName()        | `Material'/Game/Meshes/Test/Material.Material'` |                                              |

## AssetData Related

| AssetData Related        |                                         | note                     |
| :------------------------ | --------------------------------------- | ------------------------ |
| original                 | `'/Game/Meshes/Test/Material.Material'` |                          |
| **AssetData.ObjectPath** | `/Game/Meshes/Test/Material.Material`   |                          |
| AssetData.PackageName    | `/Game/Meshes/Test/Material`            |                          |
| AssetData.PackagePath    | `/Game/Meshes/Test`                     |                          |
| AssetData.AssetName      | `Material`                              |                          |
| AssetData.AssetClass     | `Material`                              | `Blueprint` if it's a BP |

## FPackageName Related 

| FPackageName Related                           |                                                                                    |     |
| :------------------------------------------- | ---------------------------------------------------------------------------------- | --- |
| original                                    | `/Game/Mesh/SM_MatPreviewMesh_01.SM_MatPreviewMesh_01:SubObjectTest`               |     |
| FPackageName::GetLongPackagePath            | `/Game/Mesh`                                                                       |     |
| FPackageName::GetLongPackageAssetName       | `SM_MatPreviewMesh_01.SM_MatPreviewMesh_01:SubObjectTest`                          |     |
| FPackageName::GetShortName                  | `SM_MatPreviewMesh_01.SM_MatPreviewMesh_01:SubObjectTest`                          |     |
| **FPackageName::LongPackageNameToFilename** | maybe like`../../../../../test_proj/ToolExample/Content/Mesh/SM_MatPreviewMesh_01` |     |
| UAssetToolsImpl::CreateUniqueAssetName()    | args in `/Game/Cube`,`_suffix`, args out `/Game/Cube_suffix_x`,`Cube_suffix_x`     |     |


There are also some utility functions for converting betwwen `SystemPath` and `VirtualPath` in `FPackageName`.
- `static bool TryConvertFilenameToLongPackageName`
- `static bool TryConvertLongPackageNameToFilename`
- and many others
- Unless you know 100% what you're doing, **DO NOT** manipulate Unreal assets through system APIs. Unreal knows nothing about what happens to assets and the reference graphs it tracks may be broken.

## FSoftObjectPath Related

| FSoftObjectPath Related | ||
| :----------------------- |- | -|
| original                | `/Game/Mesh/SM_MatPreviewMesh_01.SM_MatPreviewMesh_01:SubObjectTest` |  |
| GetLongPackageName()    | `/Game/Mesh/SM_MatPreviewMesh_01`| |
| GetSubPathString()      | `SubObjectTest`| SubObject is rare in UE nowadays so it's optional and often empty. |
| GetAssetPathString()    | `/Game/Mesh/SM_MatPreviewMesh_01.SM_MatPreviewMesh_01`||
| GetAssetNameString()    | `SM_MatPreviewMesh_01` | there are also `FName` function variants                           |