
---
title: "UE4中的断言"
date: 2022-07-17T15:15:59+08:00
draft: false
# tags: [ "" ]
categories: [ "UE"]
# keywords: [ ""]
# lastmod: 2022-07-17T15:15:59+08:00
# CJKLanguage: Chinese, Japanese, Korean
isCJKLanguage: true
slug: "1e49fb86"
toc: false
mermaid: false
# latex support
# katex: true
# markup: mmark
# mmarktoc: false 
UEVersion: 4.26.2
---

# 编译flags

从`Engine\Source\Runtime\Core\Public\Misc\Build.h`中定义了一系列的编译宏(这些宏的内容应该可以在UBT里重新定义)。

和断言有关的包括

| 宏              | 作用                                 |
| --------------- | ------------------------------------ |
| `DO_GUARD_SLOW` | 编译checkSlow,checkfSlow和verifySlow |
| `DO_CHECK`      | 编译check族和verify族函数            |
| `DO_ENSURE`     | 编译ensure族函数                     |

其中

- Debug模式

DEBUG模式下所有的断言默认开启
```
#define DO_GUARD_SLOW 1 
#define DO_CHECK 1
#define DO_ENSURE 1
```

- Development模式

Development下，SLOW的被禁用(不懂为什么起名叫SLOW，起名叫DEBUG不好吗)。


- Shipping和Test模式

Test模式下和`SHIPPING without editor`下差不多。
在Shipping模式下分为两种，如果是带`editor`的情况下说明是在调试打包，
此时定义为
```
#define DO_GUARD_SLOW 0
#define DO_CHECK 1
#define DO_ENSURE 1
```

如果没有editor则, DO_CHECK和DO_ENSURE的值与`USE_CHECKS_IN_SHIPPING`和`USE_ENSURES_IN_SHIPPING`有关。
```
#define DO_GUARD_SLOW 0
#define DO_CHECK USE_CHECKS_IN_SHIPPING
#define DO_ENSURE USE_ENSURES_IN_SHIPPING
```

`USE_CHECKS_IN_SHIPPING`和`USE_ENSURES_IN_SHIPPING`这两个值有默认值，默认情况下是禁用的

```
#define USE_CHECKS_IN_SHIPPING 0
#define USE_ENSURESIN_SHIPPING USE_CHECKS_IN_SHIPPING
```

也就是在发行版本下，默认`check,verify,ensure`都是禁用的。

# 断言种类

- Check系列

`check`系列类似于std里的`assset`,在release版本`check`会被定义为空宏。
如果碰见false会立刻触发崩溃。

常用于检查变量。

```
checkSlow(Mesh != nullptr);//只在DEBUG下生效 
check(Mesh != nullptr);
checkf(WasDestroyed, TEXT( "Failed to destroy Actor %s (%s)"), *Actor->GetClass()->GetName(), *Actor->GetActorLabel());//check第一个条件为真，否则格式化打印后面的信息
```

还有一些额外的，`CheckNoEntry`约等于`asset(0)`，表示这段代码永远不该被访问到。
`CheckNoReentry`会在此处定义一个变量，当第二次访问的时候会abort

- Verify系列

Verify系列用法类似于`DX11`里常用的`HR`宏，它用来检查函数的返回值。
在发行版本里尽管检查部分被删除,但是函数体仍然执行。

```
verify((Mesh = GetRenderMesh()) != nullptr); // 无论任何情况，Mesh都会被赋值
```

- Ensure系列
ensure系列执行检查，但是不会abort，只会打印错误信息。

```
ensureMsg(Node != nullptr, TEXT("Node is invalid"));//当Node为空的时候会打印当前堆栈，并打印信息，但是代码会继续执行
```


# Check断言实现:

`check`宏实质上是执行`FDebug::CheckVerifyFailed()` 函数

 ```
 	#define UE_CHECK_IMPL(expr) \
		{ \
			if(UNLIKELY(!(expr))) \   //如果expr为真的话那么什么也不会发生
			{ \
				struct Impl \
				{ \
					static void FORCENOINLINE UE_DEBUG_SECTION ExecCheckImplInternal() \
					{ \
						FDebug::CheckVerifyFailed(#expr, __FILE__, __LINE__, TEXT("")); \
					} \
				}; \
				Impl::ExecCheckImplInternal(); \ // 有调试器的话这里会触发调试器中断，没有调试器在这里就会abort
				PLATFORM_BREAK(); \ //触发调试器中断，如果挂了调试器，在MSVC下是__debugbreak()。 不懂这一行作用
				CA_ASSUME(false); \
			} \
		}
 ```
 
 跟进`ExecCheckimplInternal`函数进去，主要是检查格式化字符串是否正确，并且打印调用栈。
 跟到 `AssetFailedImplV`以后
 会发现打印error字符串
 
 ```
 	if (GError)
	{
		GError->Logf(TEXT("Assertion failed: %s") FILE_LINE_DESC TEXT("\n%s\n"), ErrorString, ANSI_TO_TCHAR(File), Line, DescriptionString);
	}
 ```
 
 
 ![UE4中的断言-2022-07-17-15-17-37](https://img.blurredcode.com/img/UE4中的断言-2022-07-17-15-17-37.png?x-oss-process=style/compress)
 
 `GError`全局变量的定义如下
 ```
/** Critical errors. */
CORE_API FOutputDeviceError* GError = NULL;
```
它是继承了`FOutputDevice` 这个基类，拥有打印的功能，同时他是负责打印致命错误的。

![UE4中的断言-2022-07-17-15-17-54](https://img.blurredcode.com/img/UE4中的断言-2022-07-17-15-17-54.png?x-oss-process=style/compress)

`Logf`函数会转而调用`LogfImpl`函数，并实质上调用`Serialize`函数，这个函数是个纯虚函数，分派到子类实现。
```
	virtual void Serialize( const TCHAR* V, ELogVerbosity::Type Verbosity, const FName& Category ) = 0;
```
在Windows上, 其应该调用`FWindowsErrorOutputDevice::Serialize`。

里面逻辑比较复杂，首先调用windows api检查错误，`::GetLastError`。

然后在`GIsGuarded`，也就是在`WinMain`函数里的try-catch里发生的话，就会首先检查是GPU错误还是CPU错误，CPU错误的话会触发

```
		if (GIsGPUCrashed)
		{
			ReportGPUCrash(Msg, NumStackFramesToIgnore);
		}
		else
		{
			ReportAssert(Msg, NumStackFramesToIgnore);//CPU crash, asset
		}
```

而进到`ReportAssert`里就是收集调用栈并打印

```
FORCENOINLINE void ReportAssert(const TCHAR* ErrorMessage, int NumStackFramesToIgnore)
{
	/** This is the last place to gather memory stats before exception. */
	FGenericCrashContext::SetMemoryStats(FPlatformMemory::GetStats());


	//  在打印的调用栈里 忽略`ReportAssert`和`RaiseException`两个函数
	FAssertInfo Info(ErrorMessage, NumStackFramesToIgnore + 2); // +2 for this function and RaiseException()

	ULONG_PTR Arguments[] = { (ULONG_PTR)&Info }; //把调用栈传递给 exception-handler
	//https://docs.microsoft.com/en-us/windows/win32/api/errhandlingapi/nf-errhandlingapi-raiseexception
	//抛异常
	::RaiseException(AssertExceptionCode, 0, UE_ARRAY_COUNT(Arguments), Arguments); //win32 API

```

抛异常以后逻辑被`WinMain`里的exception-handler捕获，可以看

```
//winMain
#if !PLATFORM_SEH_EXCEPTIONS_DISABLED
		__try
#endif
 		{
			GIsGuarded = 1;
			// Run the guarded code.
			ErrorLevel = GuardedMainWrapper( CmdLine );  //  游戏的主入口，在这里面触发异常
			GIsGuarded = 0;
		}
#if !PLATFORM_SEH_EXCEPTIONS_DISABLED
		__except( GEnableInnerException ? EXCEPTION_EXECUTE_HANDLER : ReportCrash( GetExceptionInformation( ) ) )
		{
#if !(UE_BUILD_SHIPPING && WITH_EDITOR)
			// Release the mutex in the error case to ensure subsequent runs don't find it.
			ReleaseNamedMutex();
#endif
			// Crashed.
			ErrorLevel = 1;
			if(GError)
			{
				GError->HandleError();  //触发异常以后，如果GError可以打印，就跳转到GError->HandleError(), 在这里打印错误信息
			}
			LaunchStaticShutdownAfterError();
			FPlatformMallocCrash::Get().PrintPoolsUsage();
			FPlatformMisc::RequestExit( true );  // 退出程序
		}
#endif
```

重新跟进`HandleError` ,可以看到打印逻辑了。
调用堆栈的错误信息存储在`GErrorHist`这个变量里，
`TCHAR GErrorHist[16384]	= TEXT("");`
这个变量在哪里被填充的就不跟进去看了，估计在`RaiseException` 的附近。

```
	// Dump the error and flush the log.
#if !NO_LOGGING
	FDebug::LogFormattedMessageWithCallstack(LogWindows.GetCategoryName(), __FILE__, __LINE__, TEXT("=== Critical error: ==="), GErrorHist, ELogVerbosity::Error);
#endif
	GLog->PanicFlushThreadedLogs();
```

对应这一块
![UE4中的断言-2022-07-17-15-18-42](https://img.blurredcode.com/img/UE4中的断言-2022-07-17-15-18-42.png?x-oss-process=style/compress)