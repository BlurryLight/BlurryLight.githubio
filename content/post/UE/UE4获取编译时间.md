
---
title: "UE4 Get Binary Build Timestamp"
date: 2022-11-27T14:25:37+08:00
draft: false
categories: [ "UE"]
isCJKLanguage: false 
slug: "6b21234f"
toc: false
mermaid: false
fancybox: false
# latex support
# katex: true
# markup: mmark
# mmarktoc: false 
UEVersion: 4.26.2
---

{{< zhTranslation "UE4获取二进制编译时间戳" >}} 


Recently, we need a string representation of timestamps to tag the built UE4 game binary.
To test random topics I package game about ~10 versions a day, and we need a string tag to distinguish different versions.

The expected string is `v20220101T235959`, which in UE format is `v%Y%m%dT%H%M%S`.

# Get Compile Time Timestamp

The trick is that we must record the **BuiltTime** in somewhere when compilers are doing their work.
We cannot use `clock()` or any other similar methods to get the time, since they are runtime timers that return the timestamp when they are called.

After some search, I found some compiler predefined macros about the timestamp.

> Ref：[c++ - Which macro is more exact? __TIME__ or __TIMESTAMP__? - Stack Overflow](https://stackoverflow.com/questions/27691101/which-macro-is-more-exact-time-or-timestamp)

|Macro|Format|Example|
|:-|:-|:-|
|`__DATA__`|`mmm dd yyyy`|Jan 14 2012|
|`__TIME__`|`hh::mm::ss`|22:29:12|
|`__TIMESTAMP__`|`Ddd Mmm Date hh::mm::ss yyyy`|Wed Jan 18 22:29:12 2012|


Don't use `__TIMESTAMP__`, it means The date and time of *the last modification of the current source file*, which is not what we want.
> MSDN: `__TIMESTAMP__` Defined as a string literal that contains the date and time of the last modification of the current source file, in the abbreviated, constant length form returned by the CRT asctime function, for example, Fri 19 Aug 13:32:58 2016. This macro is always defined.

# Format TimeString

There are some util functions like `FDateTime::Parse` and `FDateTime::ParseHttpDate`, however neither of them can parse `mmm dd yyyy hh:mm:ss` timestamp strings.
Therefore we have to manually preprocess these macros, make them parsable by `FDateTime`, or parse ourselves and then mannually construct a `FDateTime`.

I choose the first method.
There are many ways to implement it.

- convert `mmm dd yyyy hh:mm:ss` to unix timestamp, then call `FDateTime::FromUnixTimestamp`
- convert it to `asctime`/`ISO8601`/`HTTP-date` or whatever format UE can parse, then call `FDateTime::ParseHttpDate`.


I prefer the second way because there are some handy functions in `<ctime.h>`.

`strptime` maybe the best choice to parse an arbitrary formatted timestamp, however it'a POSIX function, not part of std.
MSVC doesn't have it.
We have to use the `std::istringstream/std::get_time` instead, not a very elegent choice, because I don't like the `<*stream>` headers in the std.

The implementation is straightforward.

```cpp
	static const std::string kBuildDate(__DATE__);
	static const std::string kBuildTime(__TIME__);
	std::tm tm = {};
	std::string BuildTime = kBuildDate + " " + kBuildTime;
	UE_LOG(LogTemp,Log,TEXT("%s"),UTF8st_TO_TCHAR(BuildTime.c_str()));
	std::istringstream ss(BuildTime);
	ss >> std::get_time(&tm, "%b %d %Y %H:%M:%S");
	// not thread safe
	FString AscTime(UTF8_TO_TCHAR(std::asctime(&tm)));
	AscTime.TrimEndInline(); // remove the \n at end
	UE_LOG(LogTemp,Log,TEXT("%s"),*AscTime);
	
	// LogTemp: Time is: 2022-11-08-10-46-06
	// LogTemp: Time is: 2022.11.08-10.46.06
	// LogTemp: Time is: 2022-11-08T10:46:06.000Z
	// LogTemp: Time is: Tue, 08 Nov 2022 10:46:06 GMT
	// LogTemp: Formated Version: v20221108T104606	
	FDateTime Parse2;
	FDateTime::ParseHttpDate(*AscTime,Parse2);
	UE_LOG(LogTemp,Log,TEXT("Time is: %s"),*Parse2.ToString(TEXT("%Y-%m-%d-%H-%M-%S")));
	UE_LOG(LogTemp,Log,TEXT("Time is: %s"),*Parse2.ToString());
	UE_LOG(LogTemp,Log,TEXT("Time is: %s"),*Parse2.ToIso8601());
	UE_LOG(LogTemp,Log,TEXT("Time is: %s"),*Parse2.ToHttpDate());
    
	FString Version = TEXT("v") + Parse2.ToString(TEXT("%Y%m%dT%H%M%S"));
	UE_LOG(LogTemp,Log,TEXT("Formated Version: %s"),*Version);
```