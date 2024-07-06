
---
title: "TSubclassOf的作用与实现"
date: 2022-07-21T00:49:10+08:00
draft: false
# tags: [ "" ]
categories: [ "UE"]
# keywords: [ ""]
# lastmod: 2022-07-21T00:49:10+08:00
# CJKLanguage: Chinese, Japanese, Korean
isCJKLanguage: true
slug: "48413275"
toc: true
mermaid: true
# latex support
# katex: true
# markup: mmark
# mmarktoc: false 
UEVersion: 4.26.2
---

# TSubclassOf的作用

[UE官方文档](https://docs.unrealengine.com/4.27/zh-CN/ProgrammingAndScripting/ProgrammingWithCPP/UnrealArchitecture/TSubclassOf/)里描述了一个通过TSubclassOf在蓝图里限制下拉框选取范围的例子，不过感觉也不是讲的很清楚。

`UClass`描述了一个`UObject`类的反射信息。
通过`obj->GetClass()`能够获取其`UClass`信息。

UE中的引用分为`obj reference`和`class reference`。

一个朴素的`UClass*`可以指代任意`UObject`，所以在蓝图中选择类型的时候不好选。

```
	UPROPERTY(EditAnywhere,BlueprintReadWrite,Category="Class")
	UClass* SMBody_Class;
```

![edit-79ab670b46b24305b43bc464dfbb283c-2022-07-17-16-58-15](https://img.blurredcode.com/img/edit-79ab670b46b24305b43bc464dfbb283c-2022-07-17-16-58-15.png?x-oss-process=style/compress)

而`TSubclassOf`可以限定只引用某个类型及其Child类型的类型。
比如

```
UPROPERTY(EditAnywhere,BlueprintReadWrite,Category="Class")
TSubclassOf<ACharacter>  SMBody_Class2;
```

![edit-79ab670b46b24305b43bc464dfbb283c-2022-07-17-16-59-53](https://img.blurredcode.com/img/edit-79ab670b46b24305b43bc464dfbb283c-2022-07-17-16-59-53.png?x-oss-process=style/compress)

如果在蓝图中选中了一个`UClass`或者一个`TSubClassof`，可以在运行期根据选中的`UClass`信息动态创建Actor。

```
	{
		auto loc = FVector(000,500,0);
		auto rot = FRotator(0);
		auto info = FActorSpawnParameters();
		info.Name = FName(TEXT("BySubClass"));
		auto actor = GetWorld()->SpawnActor<ACharacter>(SMBody_Class2,loc,rot,info);
		if(actor)
			actor->SetActorLabel("BySubClass");
	}
```

# TSubclassOf的实现

`TSubclassOf`的实现位于`\Engine\Source\Runtime\CoreUObject\Public\Templates`，下面包含`Cast.h`和`SubClassOf`两个文件。
`Cast`的实现以后再研究。

`TSubclassOf`的比较重要的函数包括

```C++
template<class TClass>
class TSubclassOf
{

public:
	typedef typename TChooseClass<TIsDerivedFrom<TClass, FField>::IsDerived, FFieldClass, UClass>::Result TClassType;
	typedef typename TChooseClass<TIsDerivedFrom<TClass, FField>::IsDerived, FField, UObject>::Result TBaseType;
private:
	TClassType* Class;
public:
	/** Default Constructor, defaults to null */
	FORCEINLINE TSubclassOf() :Class(nullptr){}
	/** Constructor that takes a UClass and does a runtime check to make sure this is a compatible class */
	FORCEINLINE TSubclassOf(TClassType* From) :Class(From){}
	/** Copy Constructor, will only compile if types are compatible */
	template <class TClassA, class = decltype(ImplicitConv<TClass*>((TClassA*)nullptr))>
	FORCEINLINE TSubclassOf(const TSubclassOf<TClassA>& From) :
		Class(*From){}

	/** Assignment operator from UClass, the type is checked on get not on set */
	FORCEINLINE TSubclassOf& operator=(TClassType* From)
	{
		Class = From;
		return *this;
	}
	/** Dereference back into a UClass, does runtime type checking */
	FORCEINLINE TClassType* operator*() const
	{
		if (!Class || !Class->IsChildOf(TClass::StaticClass()))
		{
			return nullptr;
		}
		return Class;
	}
	...

```

首先从成员变量`Class`开始，其记录了`TSubclassOf`内部所保存的`UClass*`指针。
在`TSubClassof<APawn> ptr;`这样的一行中，`TClass`作为模版参数被初始化为`APawn`。
随后我们分析`TClassType`和`TBaseType`。

`TChooseClass<bCondition,typeA,typeB>`类似于三目函数，根据第一个参数布尔变量的值返回typeA或者typeB。
`TIsDerivedFrom<TClass, FField>::IsDerived`是一个巧妙的模版函数，他判断第一个模版参数TClass是否继承于第二个模版参数FField，是返回true否返回false。

这里是判断传入的模版参数类型的Class信息是继承于`FFieldClass`，还是`UClass`，其Obj信息是继承于`FField`还是`UObject`。
关于`FField`和`UObject`的分拆似乎是从`4.25`版本开始的，这部分还没有研究，目前我接触到的所有的对象都是继承于`UObject`。

所以如果传入的是`APawn`的TClass，那么其`TClassType`和`TBaseType`分别为`UClass`和`UObject`。

TSubclassOf的一段用法可以见

```C++
TSubclassOf<APawn> ptr;
ptr = UStaticMesh::StaticClass(); // StaticClass返回UClass*,被保存到ptr内部的`Class`成员变量上

//在ptr->Get()的时候执行isChildOf检查, StaticMesh不在APawn的继承链上
GWorld->SpawnActor<APawn>(ptr->Get(),.....); //ptr->Get() will be nullptr

```

`TSubclassOf`在解引用，获得其内部保存的`UClass`指针时会执行运行时检查。
会检查其内部的`UClass`成员，是否是`TSubclassOf<TClass>`种模版参数的`TClass`的子对象，不是会返回NULL。

`IsChildOf`是一个`O(N)`(似乎在发行版本里是O(1))的算法。 `A->isChildOf(B)`会沿着A的继承链一直往上寻找，逐个比对UClass信息，直到与B的UClass相等，或者找到UObject为止。

{{<mermaid>}}
graph TD
A[UObject]
A-->B[AActor]
B-->C[APawn]
A-->D[UStreamableRenderAsset]
D-->E[UStaticMesh]
{{</mermaid>}}

由于`APawn`和`UStaticMesh`不在一条继承链上，所以在需要一个APawn对象时候`SpawnActor<APawn>`，传进去一个`UStaticMesh`的指针是不会生效的。

这种设计对蓝图比较好，蓝图可以利用`TSubclassOf<APawn>`筛选出所有`APawn`类供选择。

## TChooseClass的实现

很简单的偏特化的应用

```CPP
template<bool Predicate,typename TrueClass,typename FalseClass>
class TChooseClass;

template<typename TrueClass,typename FalseClass>
class TChooseClass<true,TrueClass,FalseClass>
{
public:
	typedef TrueClass Result;
};

template<typename TrueClass,typename FalseClass>
class TChooseClass<false,TrueClass,FalseClass>
{
public:
	typedef FalseClass Result;
};

```

## TIsDerivedFrom的实现

这个实现可以说精巧,利用了编译器的函数重载和sizeof编译期求值的特性。
首先定义了两个不同大小类型，具体是什么类型不重要，只要大小不一样即可。

然后定义一个返回`DerivedType`类型的函数，函数体不重要，只需要返回这个类型即可。

最后通过`Test(DerivedTypePtr())`，实际上相当于`Test(DerivedType*)`，编译器会决定函数重载。
如果`DerivedType*`能够转换到`BaseType*`，那么此函数返回`Yes&`，否则返回`No&`。
同时通过`sizeof`比较，即可获得布尔值。

```CPP
template<typename DerivedType, typename BaseType>
struct TIsDerivedFrom
{
	// Different size types so we can compare their sizes later.
	typedef char No[1];
	typedef char Yes[2];

	// Overloading Test() s.t. only calling it with something that is
	// a BaseType (or inherited from the BaseType) will return a Yes.
	static Yes& Test( BaseType* );
	static Yes& Test( const BaseType* );
	static No& Test( ... );

	// Makes a DerivedType ptr.
	static DerivedType* DerivedTypePtr(){ return nullptr ;}

	public:
	// Test the derived type pointer. If it inherits from BaseType, the Test( BaseType* ) 
	// will be chosen. If it does not, Test( ... ) will be chosen.
	static const bool Value = sizeof(Test( DerivedTypePtr() )) == sizeof(Yes);

	static const bool IsDerived = Value;
};
```

根据我的测试`https://godbolt.org/z/v57neb1EE`，`https://godbolt.org/z/e3KzeMh76`，`TIsDerivedFrom`不能处理`volatile`类型，应该再加一个`static Yes& Test( const volatile BaseType* );`重载。

同时生成`DerivedType*`也不需要一个函数，虽然模版参数要在实例化的时候才知道，但是用模版参数声明一个指针还是做得到的，声明`DerivedType* ptr`即可。