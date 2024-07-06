
---
title: "UE4的临界区"
date: 2022-07-27T01:08:41+08:00
draft: false
# tags: [ "" ]
categories: [ "UE"]
# keywords: [ ""]
# lastmod: 2022-07-27T01:08:41+08:00
# CJKLanguage: Chinese, Japanese, Korean
isCJKLanguage: true
slug: "158130ce"
toc: true
mermaid: false
# latex support
# katex: true
# markup: mmark
# mmarktoc: false 
UEVersion: 4.26.2
---

由于UE是多线程渲染结构，所以在主线程更新某个参数，而渲染线程需要某个参数的时候，往往需要加锁。
其实C++在C++11以后提供了统一的锁结构，`std::mutex`。
不过UE还是自己实现了一套,毕竟C++的来的太迟了。
# Windows平台

UE的临界区分为3类，一类是普通的临界区，一类是`SystemWide`的，一类是读写锁。

```
typedef FWindowsCriticalSection FCriticalSection;
typedef FWindowsSystemWideCriticalSection FSystemWideCriticalSection;
typedef FWindowsRWLock FRWLock;
```

## FWindowsCriticalSection

`FWindowsCriticalSection`主要依赖`Windows`提供的`CriticalSection`完成，大致对应`pthread_mutex`功能。
注意`windows`的`mutex`和linux的完全是两码事，`windows`的mutex是系统全局的，大致对应linux系统的`named filelock`。
其他进程也能加解这把锁。

比较值得注意的处理手法是UE在`MinimalWindowsAPI.h`里重新声明了一些函数和类，而没有引入`windows.h`，大概是害怕windows.h的宏污染，而且可以自然的把一些东西包裹在`Windows`命名空间下。

```cpp
    // UE声明了一些类型,这些类型的具体实现在Windows SDK里
	// Typedefs for standard handles
	typedef void* HANDLE;
	typedef HINSTANCE__* HINSTANCE;
	typedef HINSTANCE HMODULE;
	typedef HWND__* HWND;
	typedef HKEY__* HKEY;
	typedef HDC__* HDC;
	typedef HICON__* HICON;

	typedef HICON__* HCURSOR;

    //包括声明了必要的win32 API,这些函数函数体由微软实现
	// Critical sections
	MINIMAL_WINDOWS_API void WINAPI InitializeCriticalSection(LPCRITICAL_SECTION lpCriticalSection);
	MINIMAL_WINDOWS_API BOOL WINAPI InitializeCriticalSectionAndSpinCount(LPCRITICAL_SECTION lpCriticalSection, DWORD dwSpinCount);
    ...
```
### 初始化
具体的实现没啥好说的，用了`RAII`的手法。
在构造函数初始化临界区,在析构函数销毁

```cpp
	FORCEINLINE FWindowsCriticalSection()
	{
		CA_SUPPRESS(28125);
		Windows::InitializeCriticalSection(&CriticalSection);
        // 当线程碰见临界区的时候，自旋4000 Cycles以后再sleep
        // 在4GHZ CPU上
        // 4000 / (4 * 10 ^ 9) * 10^3 约等于 1 milliseconds
		Windows::SetCriticalSectionSpinCount(&CriticalSection,4000);
	}
	FORCEINLINE ~FWindowsCriticalSection()
	{
		Windows::DeleteCriticalSection(&CriticalSection);
	}
```
### 加解锁
对应`Windows::EnterCriticalSection(&CriticalSection);`和`LeaveCriticalSection(&CriticalSection);`

提供了`tryLock`指令，在加锁失败的时候返回false。

## FWindowsSystemWideCriticalSection

全局锁，全局锁可以用来检测游戏多开。
在实现上主要利用Windows API `Mutex`。
在构造函数时创建，创建失败时候检测`isValid`。

### 初始化

```cpp
FWindowsSystemWideCriticalSection::FWindowsSystemWideCriticalSection(const FString& InName, FTimespan InTimeout)
{
    //一大堆检查，主要是MutexName不能有反斜杠\，不能超过255字符
    ...
	Mutex = CreateMutex(NULL, true, MutexName);
    // 如果已经有这把锁。。。
    if (Mutex != NULL && GetLastError() == ERROR_ALREADY_EXISTS)
    {
        // 如果InTimeOut这个参数合法，就等待上一个进程释放锁，否则直接返回，此时Mutex为Null
    }
```

一个比较合理的应用就是检查多开，见[UE4发布（打包）后单游戏实例——”锁“（防多开）](https://blog.csdn.net/KKsuser/article/details/111026394)
```cpp
//宏判断是否是在编辑器，只有打包发布才需要加锁
#if !WITH_EDITOR
	//创建一个名为 #UE4-ACTGame 的锁 因为名称是固定的，所以当我们游戏启动加载的时候会去创建这个名称的一个锁如果已经有这么一个锁了则会创建失败
	Check = new FWindowsSystemWideCriticalSection(TEXT("#UE4-ACTGame")); 
	if (Check->IsValid()) //检查这个锁是否创建成功 
	{}
	else 
	{
        //创建失败，请求关闭程序
        FGenericPlatformMisc::RequestExit(true);
	}
#endif //宏判断
```


# Unix平台 


```cpp
typedef FPThreadsCriticalSection FCriticalSection;
typedef FUnixSystemWideCriticalSection FSystemWideCriticalSection;
typedef FPThreadsRWLock FRWLock;
```

读写锁和临界区由`pthread`实现，而全局锁采用POSIX API实现。


## FPThreadCriticalSection

没什么太多的细节，使用的是`pthread`的mutex实现。
注意似乎`CriticalSection`在会比pthread_mutex慢一点，见[为什么std::mutex在windows上的开销比在linux上的大？](https://www.zhihu.com/question/31265654)，不过一般不是竞争特别激烈的时候这种开销都无关紧要。


注意`pthread_mutex`的加解锁和`CriticalSection`类似，可以在用户态完成而无需进入内核态，见[How pthread_mutex_lock is implemented](https://stackoverflow.com/questions/5095781/how-pthread-mutex-lock-is-implemented)。
只有在有竞争的时候才会进入触发上下文切换，使得竞争的进程进入睡眠。

```cpp
	FORCEINLINE FPThreadsCriticalSection(void)
	{
		// make a recursive mutex
		pthread_mutexattr_t MutexAttributes;
		pthread_mutexattr_init(&MutexAttributes);
		//允许重复加锁
		pthread_mutexattr_settype(&MutexAttributes, PTHREAD_MUTEX_RECURSIVE);
		pthread_mutex_init(&Mutex, &MutexAttributes);
		pthread_mutexattr_destroy(&MutexAttributes);
	}
```

注意`Linux`上的实现，锁是可以重复加锁的。
这个标记应该是为了统一不同平台的API表现。
因为`Windows`的`CriticalSection`默认就是可重加的，但是加解锁的次数必须匹配。
见MSDN
> After a thread has ownership of a critical section, it can make additional calls to EnterCriticalSection or TryEnterCriticalSection without blocking its execution.

## FUnixSystemWideCriticalSection

主要利用文件`handle`和`flock`实现。

构造函数里创建文件锁

```cpp
	// Attempt to open a file and then lock with flock (NOTE: not an atomic operation, but best we can do)
	// 这里应该是指打开文件和加锁不是原子操作，不像Windows的CreateMutex一步搞定
	// 有可能存在竞态条件:
	// 1. 先打开客户端A进程，再次打开客户端B进程
	// 2. A创建FileHandle,然后被进程调度
	// 3. B进程open Handle，flock
	// 4. A进程获取全局锁失败，退出。
	FileHandle = open(TCHAR_TO_UTF8(*NormalizedFilepath), O_CREAT | O_WRONLY | O_NONBLOCK, S_IRUSR | S_IWUSR | S_IRGRP | S_IWGRP | S_IROTH | S_IWOTH);
	...

	if (FileHandle != -1)
	{
		flock(FileHandle, LOCK_EX);
	}
```

在`*nix`上实现文件锁似乎有两个API，`fcntl`和`flock`。
从man手册中(`man 2 flock`)可以读到，Linux的`flock`独立实现，而`4.4BSD`的`flock`基于`fcntl`实现。

`flock`是一种建议性的锁，一个进程锁住了，其他进程依然能够读写，但是不能再加锁了，要加锁需要fd用`W`权限打开。
`flock`加的锁和打开的文件有关`struct file`，和fd的数字无关，因此采用fork或者dup的方式新增加的fd，其内部的锁是同一把。

详细的介绍可以看，抛砖引玉[Linux C Flock 使用](`https://gohalo.me/post/linux-c-flock-introduce.html`)

`flock`可以重复加锁，简单的测试可以见

```c
#include <unistd.h>
#include <stdlib.h>
#include <sys/file.h>
#include <stdio.h>
#include <assert.h>

int main(void)
{
    int ret, fd;
    setvbuf(stdout, NULL, _IONBF, 0);
    fd = open("test_flock", O_RDWR | O_CREAT | O_NONBLOCK);
    printf("Ready for Lock\n");
    ret = flock(fd, LOCK_EX);
    assert(ret == 0);
    printf("Lock Once\n");
    ret = flock(fd, LOCK_EX);
    assert(ret == 0);
    printf("Lock Twice\n");
    // sleep for 10 s
    sleep(10);
    printf("Ready for Unlock\n");
    ret = flock(fd,LOCK_UN);
    assert(ret == 0);
    printf("UnLock Once\n");
    ret = flock(fd,LOCK_UN);
    assert(ret == 0);
    printf("UnLock Twice\n");
	return 0;
}

```