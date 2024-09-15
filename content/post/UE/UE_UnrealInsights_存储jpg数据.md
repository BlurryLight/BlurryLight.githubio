
---
title: "UE | UnrealInsights 截图采用Jpg数据"
date: 2024-09-15T12:54:54+08:00
draft: false
categories: [ "UE"]
isCJKLanguage: true
slug: "e1141481"
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

背景: 我们在录制Unreal Insights时为了方便查看性能热点时候正在做什么，开了一个后台线程在定时截屏并传输数据给insights存储到Screenshot Channel里。但是最近注意到这样生成的utrace文件体积增长的非常快，很不利于存储的和分发，调查了一下原因。

# TraceScreenshot接口

`FTraceScreenshot::TraceScreenshot`接受一个未压缩过的`TArray<FColor>`或者`TArray<FLinearColor>`作为原始数据，内部会根据数据类型选择不同的压缩格式。

- FColor: PNG
- FLinearColor: exr

默认情况下FColor是压缩成PNG，压缩效率有点低，一张`1280x720`的数据要增长1M多。

```c
void ImageUtilsCompressImageArrayWrapper(int32 ImageWidth, int32 ImageHeight, const TArrayView64<const FColor>& SrcData, TArray64<uint8>& DstData)
{
	FImageUtils::PNGCompressImageArray(ImageWidth, ImageHeight, SrcData, DstData);
}
```

这里可以改一下，改成JPG。

UnrealInsights程序在解码utrace里编码的图片数据时，会调用`DecompressImage`函数来解码Blob数据。这个函数里面会根据每个格式的Magic Header来探测图片格式，再解码。

不同图片格式的`Magic Header`可以在以下链接找到。

> 参考：[File Magic Numbers](https://gist.github.com/leommoore/f9e57ba2aa4bf197ebc5)

因此只要是虚幻`ImageWrapper`模块支持的格式都可以被`encode`到utrace里。


![edit-47a77b3d8654448d852abc6d21a5a815-2024-09-13-12-03-03](https://img.blurredcode.com/img/edit-47a77b3d8654448d852abc6d21a5a815-2024-09-13-12-03-03.png?x-oss-process=style/compress)


调整格式以后再下调了一点分辨率，这样一张截图只增长70k左右的空间，基本不会造成太大的影响了。
