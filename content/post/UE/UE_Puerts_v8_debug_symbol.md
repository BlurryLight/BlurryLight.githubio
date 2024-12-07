
---
title: "Puerts 编译带调试符号的debug版v8"
date: 2023-12-23T15:18:42+08:00
draft: false
categories: ["UE", "PuerTs"]
isCJKLanguage: true
slug: "964342de"
toc: false
mermaid: false
fancybox: false
blueprint: false
# latex support
# katex: true
# markup: mmark
# mmarktoc: false 
---


最近一段时间一直在和PuerTs打交道，有的时候碰见问题想跟到v8里看一下v8的具体实现细节。
Puerts的官方提供编译好的二进制，开箱即用。
从官方下下来的v8是release版，但是压缩包里带了个`pdb`，本来以为是可以直接挂上调试符号的，但是用VS的调试器挂上pdb以后仍然没法单步进去，只能反汇编调试，跟着v8一些简单的符号调，有点痛苦。

翻了一下也不只是我碰见了个这个问题

> 参考：[v8.dll.pdb的符号文件无法加载 · Issue #7 · puerts/backend-v8](https://github.com/puerts/backend-v8/issues/7)

深入调查了一下，原因可能是因为编译v8的时候用的`Release`版，而且看样子还加了`strip_debug_info`之类的标记，对MSVC的pdb和谷歌的gn 这套不熟悉，不太确定到底是哪个标记导致的。

```bat
call gn gen out.gn\x64.release -args="target_os=""win"" target_cpu=""x64"" v8_use_external_startup_data=false v8_enable_i18n_support=false is_debug=false is_clang=false strip_debug_info=true symbol_level=0 v8_enable_pointer_compression=false is_component_build=true"
```

反正有现成的编译脚本，干脆自己编译一个debug版本好了。

fork了一份新的编译脚本，对编译9.4的脚本改了一些参数。 

[backend-v8-ci/windows_64MD_DLL_94_debug.cmd at master · BlurryLight/backend-v8-ci](https://github.com/BlurryLight/backend-v8-ci/blob/master/windows_64MD_DLL_94_debug.cmd)


并且用`github actions`也跑通了，可以白嫖github的服务来编译了。见 https://github.com/BlurryLight/backend-v8-ci/actions/runs/7301759685


如果需要 安卓/IOS/Windows静态库的可以照着上面的编译脚本酌情改下。