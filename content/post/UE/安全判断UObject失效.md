
---
title: "使用FWeakObjectPtr安全判断UObject失效"
date: 2024-06-29T13:00:20+08:00
draft: false
categories: [ "UE"]
isCJKLanguage: true
slug: "a57858d2"
toc: false
mermaid: false
fancybox: false
blueprint: false
# latex support
# katex: true
# markup: mmark
# mmarktoc: false 
---

{{% notice info %}}
Engine Version: 5.3.2
{{% /notice %}}

最近碰到一个比较奇葩的问题，就是渲染线程里绑定材质Material Uniform的地方居然保存了`UTexture*`的裸指针。

具体可以见
`Engine\Source\Runtime\Engine\Private\Materials\MaterialInstanceSupport.h`

![安全判断UObject失效-2024-06-29-13-06-05](https://img.blurredcode.com/img/安全判断UObject失效-2024-06-29-13-06-05.png?x-oss-process=style/compress)

然后在切换地图的时候，有可能因为切地图UE强杀Actor的原因导致Actor动态创建的`UTexture*`被回收，这里保存的指针就成了悬空指针。(具体的成因我还没想清楚，因为理论上来说Material那里也记录了Texture的引用，不过现象就是在切地图的时候有几率碰见崩溃)。

具体崩溃的堆栈在于`void FUniformExpressionSet::FillUniformBuffer`

仔细阅读这段代码,似乎`Epic`也发现这里可能会产生悬空指针，只是他们也没弄明白这块到底是怎么产生的...我翻了一下UDN，也有人碰见过类似的问题，不过他那个原因是因为他绑定的UTexture忘记了加`UProperty`,导致被GC回收了，和我这还不太一样...我这里UTexture明明放在蓝图的变量怎么都被回收了..

![安全判断UObject失效-2024-06-29-13-09-38](https://img.blurredcode.com/img/安全判断UObject失效-2024-06-29-13-09-38.png?x-oss-process=style/compress)

# IsValid / IsValidLowLevel 是否有效


IsValid在蓝图里被常常用来检查一个UObject是否有效。
![安全判断UObject失效-2024-06-29-13-11-07](https://img.blurredcode.com/img/安全判断UObject失效-2024-06-29-13-11-07.png?x-oss-process=style/compress)

但是很遗憾，这个方法只能被用来检查

- 空指针
- 正在被GC的对象
- 一个UProperty并且已经被GC的对象(GC系统对于Uproperty会自动置空)

对于**悬空指针**的情况，无论是`IsValid/IsValidLowLevel`都无法进行安全的检查，甚至会立刻崩溃。
原因是`IsValid / IsValidLowLevel`会试图访问这个UObject内部的成员,但是如果是悬空指针，所处的内存读到的是垃圾数据

```cpp
bool UObjectBase::IsValidLowLevel() const
{
	if( this == nullptr )
	{
		UE_LOG(LogUObjectBase, Warning, TEXT("NULL object") );
		return false;
	}
	if( !ClassPrivate )  // 试图访问成员变量，可能会访问到不可访问的内存，或者调用到不正确的方法
	{
		UE_LOG(LogUObjectBase, Warning, TEXT("Object is not registered") );
		return false;
	}
	return GUObjectArray.IsValid(this);
}
```



# 正确检查UObject是否是悬空指针


虚幻提供了一个`FWeakObjectPtr`用来正确检查悬空指针，其原理是一个胖指针，额外记录了一些UObject的信息用来判断是否失效。

```cpp
void FWeakObjectPtr::operator=(const class UObject *Object)
{
	if (Object // && UObjectInitialized() we might need this at some point, but it is a speed hit we would prefer to avoid
		)
	{
		ObjectIndex = GUObjectArray.ObjectToIndex((UObjectBase*)Object);
		ObjectSerialNumber = GUObjectArray.AllocateSerialNumber(ObjectIndex);
		checkSlow(SerialNumbersMatch());
	}
	else
	{
		Reset();
	}
}
```

当`FWeakObjectPtr`从一个UObject*初始化时，其成员变量会记录这个UObject的唯一索引。
再利用`UObject`都会被注册到`GUObjectArray`这个特性。
只要判断当前`GUObjectArray`里是否还有这个索引，就可以判断这个UObject是否有效。

具体可以看`FORCEINLINE FUObjectItem* Internal_GetObjectItem() const`方法的实现，其主要的探测方法即是判断当前`GUObjectArray`是否还有这个Uobject。

另外一个很好的性质就是`FWeakObjectPtr`可以在其他线程探测`UObject`是否有效( `FWeakObjectPtr::IsValid`函数有一个额外参数`bThreadSafeTest`)，因为它内部只有对`GUObjectArray`的读操作，没有写操作。

# 修复方案

回到最开始的问题，最后考虑了一下，把保存材质参数绑定的地方额外保存一个`FWeakObjectPtr`，并在其他一些设置材质参数的地方和获取材质参数的地方额外添加了若干行`if constexpr(std::is_point_v(ValueType))`。

如果是`UObject*`类型的参数，就需要额外初始化这个`FWeakObjectPtr`，并且在获取参数并绑定到`Uniform`阶段，也需要额外检查`FWeakObjectPtr`的有效性。

![安全判断UObject失效-2024-06-29-13-33-37](https://img.blurredcode.com/img/安全判断UObject失效-2024-06-29-13-33-37.png?x-oss-process=style/compress)