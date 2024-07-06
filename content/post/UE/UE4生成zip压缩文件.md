
---
title: "UE4生成zip压缩文件"
date: 2022-11-20T16:39:41+08:00
draft: false
categories: [ "UE"]
isCJKLanguage: true
slug: "b7ea1d63"
toc: false
mermaid: false
fancybox: false
# latex support
# katex: true
# markup: mmark
# mmarktoc: false 
UEVersion: 4.26.2
---


最近有用UE生成zip文件的需求。

# FZipArchiveWriter

UE提供一个非常简单的`RAII`风格的ZIP类`FZipArchiveWriter`，只有三个API（Engine/Source/Developer/FileUtilities/Public/FileUtilities/ZipArchiveWriter.h）

```cpp
	FZipArchiveWriter(IFileHandle* InFile);
	~FZipArchiveWriter();
	void AddFile(const FString& Filename, const TArray<uint8>& Data, const FDateTime& Timestamp);
```

- 构造函数需要传入一个`InFile`指针，指明压缩文件需要保存的路径
- 通过AddFile往Zip文件里添加文件名和路径
- 析构函数自动保存

该类使用起来非常简单，但是其具有几个问题。
- 最大问题是没有实现压缩算法，只是单纯的实现了Zip的文件格式，里面保存的数据是未经压缩的。
- 模块在`Developer`目录下，该目录下的文件是不能被打包进`Shipping`版本的，见[分析 UBT 中 EULA 的内容分发限制](https://imzlp.com/posts/9050/),所以只能在编辑器下用了。


## 通过第三方库压缩

可以通过[kuba--/zip: A portable, simple zip library written in C](https://github.com/kuba--/zip) 进行压缩，使用方法见ReadMe。
三个文件可以直接集成进UE，并且是Public License，无协议污染风险。

唯一需要注意的是最好保存UTF-8，所以需要从`FString`转到UTF8做处理。
如果想要保存UTF-16字符串可能需要手动加UTF16-BOM。
大端序是`0xFE 0xFF`，小端序是`0xFF 0xFE`。

似乎对UTF-8的`Entry Name`支持有点问题，但是本身zip对这个支持就不好，最好不要用中文当文件名。
https://github.com/kuba--/zip/issues/265

```cpp
FString VeryLongText = "....";
FTCHARToUTF8 VeryLongText_UTF8(*VeryLongText);
zip_t* zip = zip_open(ZipPath_UTF8.Get(), ZIP_DEFAULT_COMPRESSION_LEVEL, 'w');
{
	zip_entry_open(zip, TCHAR_TO_UTF8(TEXT("VeryLongText.txt")));
	{
		zip_entry_write(zip, VeryLongText_UTF8.Get(), VeryLongText_UTF8.Length());
	}
	zip_entry_close(zip);
}
zip_close(zip);
```
