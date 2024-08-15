
---
title: "UE5 | ChildActorComponent Transient Child在PIE下消失的问题"
date: 2024-08-15T22:49:58+08:00
draft: false
categories: [ "UE"]
isCJKLanguage: true
slug: "e488f1cb"
toc: true
mermaid: false
fancybox: false
blueprint: false
# latex support
# katex: true
# markup: mmark
# mmarktoc: false 
UEVersion: 5.3.2 
---


# To be Transient or Not To be?


```cpp
/**
    * Should the spawned actor be marked as transient?
    * @note The spawned actor will also be marked transient if this component or its owner actor are transient, regardless of the state of this flag.
    */
UPROPERTY(EditDefaultsOnly, Category=ChildActorComponent)
uint8 bChildActorIsTransient:1;
```

这个Flag主要作用是给Spawn出来的ChildActor加上Transient标记。

带上Transient标记，所有序列化和SavePackage之类的函数都会跳过这个ChildActor。
适合ChildActor可能会进行一些会导致PackageDirty的情况。


# Bug: PIE下ChildActor消失?

如果带上Transient，摆在场景上的ChildActorComponent在PIE下没法看见创建的ChildActor.


![edit-5809e22784b84ca4a70b2282f98296b6-2024-08-15-15-44-40](https://img.blurredcode.com/img/edit-5809e22784b84ca4a70b2282f98296b6-2024-08-15-15-44-40.png?x-oss-process=style/compress)


|Editor|PIE|
|-|-|
|![edit-5809e22784b84ca4a70b2282f98296b6-2024-08-15-15-45-58](https://img.blurredcode.com/img/edit-5809e22784b84ca4a70b2282f98296b6-2024-08-15-15-45-58.png?x-oss-process=style/compress)|![UE_ChildActorComponent_Transient_Bug_In_PIE-2024-08-15-22-57-15](https://img.blurredcode.com/img/UE_ChildActorComponent_Transient_Bug_In_PIE-2024-08-15-22-57-15.png?x-oss-process=style/compress)
|



仔细分析了一波原因，最后定位到在`ULevel::Serialize`里:

对于带有`Transient`标记的Actor不会被序列化到`Actors`数组里。

```cpp
	else if (Ar.IsSaving() && Ar.IsPersistent())
	{
		UPackage* LevelPackage = GetOutermost();
		TArray<AActor*> EmbeddedActors;
		EmbeddedActors.Reserve(Actors.Num());

		Algo::CopyIf(Actors, EmbeddedActors, [&](AActor* Actor)
		{
			if (!Actor)
			{
				return false;
			}

			check(Actor->GetLevel() == this);

			if (Actor->HasAnyFlags(RF_Transient))
			{
				return false;
			}

```


当从Editor进入到PIE时候，会进行`UWorld`的复制，`ULevel`的重新初始化，最后进入到

```cpp
bool ULevel::IncrementalRegisterComponents(bool bPreRegisterComponents, int32 NumComponentsToUpdate, FRegisterComponentContext* Context)
{
	// Find next valid actor to process components registration

	if (OwningWorld)
	{
		OwningWorld->SetAllowDeferredPhysicsStateCreation(true);
	}

	while (CurrentActorIndexForIncrementalUpdate < Actors.Num())
    {
        ....
        Actor->IncrementalRegisterComponents(NumComponentsToUpdate, Context);
    }
```


由于`Editor`的`ChildActor`没有被序列化到`ULevels Actor`里，新建的PIE UWorld的Level将不会有这个ChildActor，所以这里会跳过Child Actor的Register。

所以，`ChildActor`虽然存在在PIE World(由UWorld的复制的时候带过来的,所以能正常执行BeginPlay / TickFunction)，但是由于没有注册所以没有创建`RenderState`和`PhysicsState`，所以在渲染时看不到，物理上也无法交互。

一个简单的修复就是在 `ChildActorComponent`的`OnRegister`中，如果发现`ChildActor`没注册，重新注册。



```
diff --git a/Engine/Source/Runtime/Engine/Private/Components/ChildActorComponent.cpp b/Engine/Source/Runtime/Engine/Private/Components/ChildActorComponent.cpp
--- a/Engine/Source/Runtime/Engine/Private/Components/ChildActorComponent.cpp
+++ b/Engine/Source/Runtime/Engine/Private/Components/ChildActorComponent.cpp

@@ -102,6 +102,13 @@
void UChildActorComponent::OnRegister()
{
	Super::OnRegister();

    ....
 	{
 		CreateChildActor();
 	}
+   // register all components if necessary
+	if(ChildActor && !ChildActor->HasActorRegisteredAllComponents())
+	{
+		ChildActor->RegisterAllComponents();
+	}
 }
 
 void UChildActorComponent::Serialize(FArchive& Ar)

```