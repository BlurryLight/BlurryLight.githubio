
---
title: "UE | 解决Rider在Unreal Android配置下大片飘红的问题"
date: 2024-09-15T20:35:43+08:00
draft: false
categories: [ "UE"]
isCJKLanguage: true
slug: "8739b0fb"
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


# Rider打开Unreal Android 大片爆红

Rider在平时开发虚幻的时候很好用，但是一旦用来开发和调试Android平台的虚幻代码时，就会出现大片红色的情况，这是因为Rider无法正确解析Android平台的头文件路径。
我们平时的工作流基本上是在Windows上开发，然后交叉编译到Android平台，但是虚幻的UBT没有正确在Android平台导出对应NDK的头文件路径，导致Rider无法正确解析交叉编译的头文件路径。

![edit-fc42cbd22fca435288e86c43a3b38c4a-2024-09-15-18-25-50](https://img.blurredcode.com/img/edit-fc42cbd22fca435288e86c43a3b38c4a-2024-09-15-18-25-50.png?x-oss-process=style/compress)

![edit-fc42cbd22fca435288e86c43a3b38c4a-2024-09-15-18-27-15](https://img.blurredcode.com/img/edit-fc42cbd22fca435288e86c43a3b38c4a-2024-09-15-18-27-15.png?x-oss-process=style/compress)


Jetbrains的Issue里也有人提了这个问题，UBT只导出了各个模块的头文件和引擎的头文件路径，但是没有导出Android的系统库的索引路径，导致大片大片的找不到头文件的问题。

```c
"EnvironmentIncludePaths":[
      "C:\\Program Files\\Epic Games\\UE_4.26\\Engine\\Source",
      "C:\\NVPACK\\android-sdk-windows\\ndk\\21.1.6352462\\sources\\android\\native_app_glue",
      "C:\\NVPACK\\android-sdk-windows\\ndk\\21.1.6352462\\sources\\android\\cpufeatures"
   ]
```

> 参考：[UnrealBuildTool dumps incorrect EnvironmentIncludePaths for Android configuration : RIDER-54778](https://youtrack.jetbrains.com/issue/RIDER-54778/UnrealBuildTool-dumps-incorrect-EnvironmentIncludePaths-for-Android-configuration)


# 修复方案: 手动导出Android系统库的头文件路径

我们注意到虚幻还是导出了`ndk`的部分头文件路径，但是缺少了llvm标准库和交叉编译环境的`/usr/include`路径，我们只需要依葫芦画瓢导出对应的路径即可。

`Engine/Source/Programs/UnrealBuildTool/Platform/Android/UEBuildAndroid.cs`

![UE_解决Rider开发Android时大片飘红的问题-2024-09-15-20-44-27](https://img.blurredcode.com/img/UE_解决Rider开发Android时大片飘红的问题-2024-09-15-20-44-27.png?x-oss-process=style/compress)


# 问题2： Rider找不到__cplusplus定义

这个是Reshaper引擎和UBT的共同作用导致的。

> 参考：[__cplusplus has incorrect value in Unreal Engine solutions : RIDER-106463](https://youtrack.jetbrains.com/issue/RIDER-106463/cplusplus-has-incorrect-value-in-Unreal-Engine-solutions)


具体成因因为Reshaper是闭源的也不太清楚，总而言之从现象上来看，Rider的Resharper似乎带着错误的`__cplusplus`宏定义去解析libcxx的头文件，导致一些`static_assert`,`constexpr`之类的关键词解析的不对，导致大片的爆红。

这个问题也好解决，既然自动的cpp宏不对，我们就挨个手动定义，因为我们在UBT里是可以拿到每个Module是用什么cpp标准编译的。

![UE_解决Rider开发Android时大片飘红的问题-2024-09-15-20-49-18](https://img.blurredcode.com/img/UE_解决Rider开发Android时大片飘红的问题-2024-09-15-20-49-18.png?x-oss-process=style/compress)


解决问题以后，Rider就可以正常的跳转Android和Java相关的头文件了，可以愉快的魔改引擎了。

![UE_解决Rider开发Android时大片飘红的问题-2024-09-15-20-50-06](https://img.blurredcode.com/img/UE_解决Rider开发Android时大片飘红的问题-2024-09-15-20-50-06.png?x-oss-process=style/compress)



# Visual Studio ?

VS也有一样的问题，每次用AGDE调试Android的时候都是一片红，简直辣眼睛。
不过思路应该是一样的，把对应Android平台的头文件索引路径加入到`sln`的配置里，应该也可以解决这个问题。
不过我平时开发不用VS就算了，不研究了。


# Patch文件


```patch
From 84c161c6f9f285e1f8a5445af44fcd05bec8f648 Mon Sep 17 00:00:00 2001
From: ravenzhong
Date: Sun, 15 Sep 2024 20:33:20 +0800
Subject: [PATCH] fix android rider index error

---
 .../Platform/Android/AndroidToolChain.cs       |  7 +++++++
 .../Platform/Android/UEBuildAndroid.cs         |  7 +++++++
 .../ProjectFiles/Rider/RiderProjectFile.cs     | 18 ++++++++++++++++++
 3 files changed, 32 insertions(+)

diff --git a/Engine/Source/Programs/UnrealBuildTool/Platform/Android/AndroidToolChain.cs b/Engine/Source/Programs/UnrealBuildTool/Platform/Android/AndroidToolChain.cs
index a87fea12b..6de38e974 100644
--- a/Engine/Source/Programs/UnrealBuildTool/Platform/Android/AndroidToolChain.cs
+++ b/Engine/Source/Programs/UnrealBuildTool/Platform/Android/AndroidToolChain.cs
@@ -93,6 +93,10 @@ namespace UnrealBuildTool
 		public string? NDKToolchainVersion;
 		public UInt64 NDKVersionInt;
 
+
+		public string SysrootPath; //++ravenzhong
+		public string SysrootIncludePath; //++ravenzhong
+		
 		int ClangVersionMajor = -1;
 		int ClangVersionMinor = -1;
 		int ClangVersionPatch = -1;
@@ -221,6 +225,9 @@ namespace UnrealBuildTool
 
 			string GCCToolchainPath = Path.Combine(NDKPath, @"toolchains/llvm", ArchitecturePath);
 			string SysrootPath = Path.Combine(NDKPath, @"toolchains/llvm", ArchitecturePath, "sysroot").Replace("\\", "/");
+			
+			this.SysrootPath = SysrootPath; //++ravenzhong
+			this.SysrootIncludePath= Path.Combine(SysrootPath, @"usr/include"); //++ravenzhong
 
 			// toolchain params (note: use ANDROID=1 same as we define it)
 			ToolchainLinkParamsArm64 = " --target=aarch64-none-linux-android" + NDKApiLevel64Int + " --gcc-toolchain=\"" + GCCToolchainPath + "\" --sysroot=\"" + SysrootPath + "\" -DANDROID=1";
diff --git a/Engine/Source/Programs/UnrealBuildTool/Platform/Android/UEBuildAndroid.cs b/Engine/Source/Programs/UnrealBuildTool/Platform/Android/UEBuildAndroid.cs
index c445e936a..cdaa3c8db 100644
--- a/Engine/Source/Programs/UnrealBuildTool/Platform/Android/UEBuildAndroid.cs
+++ b/Engine/Source/Programs/UnrealBuildTool/Platform/Android/UEBuildAndroid.cs
@@ -455,6 +455,13 @@ namespace UnrealBuildTool
 
 			CompileEnvironment.SystemIncludePaths.Add(DirectoryReference.Combine(NdkDir, "sources/android/native_app_glue"));
 			CompileEnvironment.SystemIncludePaths.Add(DirectoryReference.Combine(NdkDir, "sources/android/cpufeatures"));
+			//++ravenzhong
+			CompileEnvironment.SystemIncludePaths.Add(DirectoryReference.Combine(NdkDir, "sources/cxx-stl/llvm-libc++/include"));
+			CompileEnvironment.SystemIncludePaths.Add(DirectoryReference.Combine(NdkDir, "sources/cxx-stl/system/include"));
+			CompileEnvironment.SystemIncludePaths.Add(DirectoryReference.Combine(NdkDir, "sources/third_party/vulkan/src/include"));
+			CompileEnvironment.SystemIncludePaths.Add(new DirectoryReference(ToolChain.SysrootIncludePath));
+			
+			//--ravenzhong
 
 			//@TODO: Tegra Gfx Debugger - standardize locations - for now, change the hardcoded paths and force this to return true to test
 			if (UseTegraGraphicsDebugger(Target))
diff --git a/Engine/Source/Programs/UnrealBuildTool/ProjectFiles/Rider/RiderProjectFile.cs b/Engine/Source/Programs/UnrealBuildTool/ProjectFiles/Rider/RiderProjectFile.cs
index b0ff096de..1e2466b57 100644
--- a/Engine/Source/Programs/UnrealBuildTool/ProjectFiles/Rider/RiderProjectFile.cs
+++ b/Engine/Source/Programs/UnrealBuildTool/ProjectFiles/Rider/RiderProjectFile.cs
@@ -589,6 +589,24 @@ namespace UnrealBuildTool
 			{
 				Writer.WriteValue(Definition);
 			}
+			//++ravenzhong
+			//https://youtrack.jetbrains.com/issue/RIDER-106463/cplusplus-has-incorrect-value-in-Unreal-Engine-solutions
+			if (Target.Platform == UnrealTargetPlatform.Android)
+			{
+				if (Target.Rules.CppStandard >= CppStandardVersion.Cpp20)
+				{
+					Writer.WriteValue("__cplusplus=202002L");
+				}
+				else if (Target.Rules.CppStandard >= CppStandardVersion.Cpp17)
+				{
+					Writer.WriteValue("__cplusplus=201703L");
+				}
+				else if (Target.Rules.CppStandard >= CppStandardVersion.Cpp14)
+				{
+					Writer.WriteValue("__cplusplus=201402L");
+				}
+			}
+			//--ravenzhong
 			Writer.WriteArrayEnd();
 		}
 
-- 
2.45.2.windows.1

```