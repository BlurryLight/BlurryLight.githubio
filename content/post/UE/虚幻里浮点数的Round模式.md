
---
title: "UE | 虚幻里浮点数的Round模式"
date: 2024-11-26T00:00:11+08:00
draft: false
categories: [ "UE", "PuerTs"]
isCJKLanguage: true
slug: "6565b5f8"
toc: true
mermaid: false
fancybox: false
blueprint: false
# latex support
# katex: true
# markup: mmark
# mmarktoc: false 
# UEVersion: 5.3.2 
---


# IEEE 754

[Floating-Point Rounding Mode (MIT/GNU Scheme 12.1)](https://www.gnu.org/software/mit-scheme/documentation/stable/mit-scheme-ref/Floating_002dPoint-Rounding-Mode.html)


IEEE 754允许4种浮点round模式:
- round-to-nearest 等同于十进制下的四舍五入。 一个比较典型的特征是，在`0.5`的时候，会round到最近的偶数。 比如`2.5`和`1.5`都会round到2.0。 对应Round函数
![edit-7bb95cc7f44d439ea8240cbcdc5164f7-2024-11-25-11-04-24](https://img.blurredcode.com/img/edit-7bb95cc7f44d439ea8240cbcdc5164f7-2024-11-25-11-04-24.png?x-oss-process=style/compress)

- toward-zero: 不管是负数还是正数都是直接截断尾部，比如`1.5`和`-1.5`都会round到`1.0`和`-1.0`。 对应Trunc函数

- round up: 始终朝上舍入，正数尾部阶段并进位，负数直接截断尾部。 比如`1.3`往上round到`2.0`，`-1.3`直接round到`-1.0`。 对应Ceil函数
- round down: 始终朝下舍入，正数直接截断尾部，负数尾部阶段并进位。 比如`1.3`直接round到`1.0`，`-1.3`往下round到`-2.0`。 对应Floor函数


# Cpp种类型强转时的取整策略


https://en.cppreference.com/w/cpp/language/implicit_conversion#Floating-integral_conversions

标准规定了浮点和整形之间的转换规则:

1. 如果浮点数转整形，会直接截断小数部分，比如`-1.5`会转成`-1`，`1.5`会转成`1`。 (round to zero)规则
2. 如果整形转浮点，但是浮点不能精确这个表示这个整数，那么会round到最近的浮点数(round to even原则)。比如`1`转成`float`会变成`1.0`，`-1`会变成`-1.0`。
3. cast中如果超过了范围那么是UB，双向都是UB

# 虚幻FMath中 TruncToInt / FloorToInt / CeilToInt / RoundToInt

主要是对 负数的部分的处理有区别

- FloorToInt: 无条件往负无穷取整
- Ceil ： 无条件往正无穷
- RountToInt: floor( X + X + 0.5) / 2 四舍五入,0.5时往正无穷舍入
- TruncToInt：无条件向0取整，比如-32.5 => -32, 32.5 => 32



# 虚幻一个注释错误的地方?

这里的注释认为 FMath::RoundToInt调用了floor(始终向下取整)，但是实际上算法和它这里是一样的，都是四舍五入。

但是虚幻的`RoundToInt`的实现不是`RoundToEven`，而是最近的整数(0.5始终向上舍入, -1.5舍入到-1)
这个实现和JS里的`Math.round`是一样的。 `Math.round(-1.5) = 1`

```cpp
static FORCEINLINE int32 RoundToInt32(double F)
{
    return FloorToInt32(F + 0.5);
}

FORCEINLINE FColor FLinearColor::QuantizeRound() const
{
	// Avoid FMath::RoundToInt because it calls floor()
	return FColor(
		(uint8)(0.5f + Clamp01NansTo0(R) * 255.f),
		(uint8)(0.5f + Clamp01NansTo0(G) * 255.f),
		(uint8)(0.5f + Clamp01NansTo0(B) * 255.f),
		(uint8)(0.5f + Clamp01NansTo0(A) * 255.f)
	);
}

```

# JS中Math.Round的实现

https://stackoverflow.com/a/30367267


JS中Math.Round的实现和虚幻是一样的。(src/runtime/runtime-maths.cc)

```cpp
RUNTIME_FUNCTION(Runtime_RoundNumber) {
  HandleScope scope(isolate);
  // 一些快速处理特殊值的分支，包括SMI的处理
  ....
  return *isolate->factory()->NewNumber(Floor(value + 0.5));
}
```

# cpp的roundf和roundevenf


c的round规则更奇怪，

```
Computes the nearest integer value to arg (in floating-point format), rounding halfway cases away from zero, regardless of the current rounding mode.
```

四舍五入，但是0.5时会往远离0的方向舍入，比如`round(1.5) = 2`, `round(-1.5) = -2`

```c
#include <cmath>


int main()
{
    printf("roundeven(+0.5) = %+.1f\n", roundeven(0.5));
    printf("roundf(+0.5) = %+.1f\n", roundf(0.5));

    printf("roundeven(-1.5) = %+.1f\n", roundeven(-1.5));
    printf("roundf(-1.5) = %+.1f\n", roundf(-1.5));

    printf("roundeven(-0.5) = %+.1f\n", roundeven(-0.5));
    printf("roundf(-0.5) = %+.1f\n", roundf(-0.5));
    /**
    roundeven(+0.5) = +0.0
    roundf(+0.5) = +1.0
    roundeven(-1.5) = -2.0
    roundf(-1.5) = -2.0
    roundeven(-0.5) = -0.0
    roundf(-0.5) = -1.0
    */
}
```


# Positive Zero & Negative Zero


IEEE 754规定了Zero有两个，一个是正0，一个是负0。 他们的区别在于符号位不同，但是在数学上是相等的。

比如上面的代码就产生了-0

```c
roundeven(-0.5) = -0.0
```

从浮点数的二进制表示来看，由于最高位是`sign bit`，所以会产生+0和-0。

但是不同语言里如何处理+0和-0是实现定义的..

```
// cpp
+0 == -0; // true
std::atan2(0, 0); // 0
std::atan2(0, -0); //0

//js
+0 == -0; // true
+0 === -0; // true
Object.is(+0,-0); false
Math.atan2(0, 0); // 0
Math.atan2(0, -0); // 3.1415926..
```

另外cpp和js都可以通过`-0 + 0 = +0`来使得-0转到+0


## V8的一个隐蔽的处理-0和+0的问题

这个问题最早发现于这里

- [[UE] static binding: ue.IntPoint constructor 有几率触发 invalid parameter · Issue #1910 · Tencent/puerts](https://github.com/Tencent/puerts/issues/1910)


产生问题的代码的逻辑伪代码类似于

```cpp

v8::Local<v8::Value> Value = ...;
if(Value.IsInt32())
{
    Foo()
}
else
{
    Bar()
}
```


V8的代码认为 `+0`属于`Int32`，而`-0`不属于，这样就会导致`-0`的时候会走到`Bar`里面去。

看v8的代码里，刻意排除了`IsMinusZero(value)`，不清楚具体这样设计的意图..
```cpp
bool Value::IsInt32() const {
  i::Object obj = *Utils::OpenHandle(this);
  if (obj.IsSmi()) return true;
  if (obj.IsNumber()) {
    return i::IsInt32Double(obj.Number());
  }
  return false;
}

bool IsInt32Double(double value) {
  return value >= kMinInt && value <= kMaxInt && !IsMinusZero(value) &&
         value == FastI2D(FastD2I(value));
}
```

