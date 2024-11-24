
---
title: "Puerts | v8::InternalField 大小和UStruct的包装成本"
date: 2024-11-24T06:05:08Z
draft: false
categories: [ "JavaScript", "PuerTs", "UE"]
isCJKLanguage: true
slug: "b3ae7426"
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

# SMI

v8要求存入的`void*`必须按2字节对齐，也就是最后一位必须是0
很容易想到这个是v8压缩指针要求。
进去看一下

```cpp
void v8::Object::SetAlignedPointerInInternalField(int index, void* value) {
  auto obj = Utils::OpenDirectHandle(this);
  const char* location = "v8::Object::SetAlignedPointerInInternalField()";
  if (!InternalFieldOK(obj, index, location)) return;

  i::DisallowGarbageCollection no_gc;
  Utils::ApiCheck(i::EmbedderDataSlot(i::Cast<i::JSObject>(*obj), index)
                      .store_aligned_pointer(obj->GetIsolate(), *obj, value),
                  location, "Unaligned pointer");
  DCHECK_EQ(value, GetAlignedPointerFromInternalField(index));
}

bool EmbedderDataSlot::store_aligned_pointer(IsolateForSandbox isolate,
                                             Tagged<HeapObject> host,
                                             void* ptr) {
  Address value = reinterpret_cast<Address>(ptr);
  if (!HAS_SMI_TAG(value)) return false;
  gc_safe_store(isolate, value);
  return true;
}
void EmbedderDataSlot::gc_safe_store(Address value) {
#ifdef V8_COMPRESS_POINTERS
  STATIC_ASSERT(kSmiShiftSize == 0);
  STATIC_ASSERT(SmiValuesAre31Bits());
  STATIC_ASSERT(kTaggedSize == kInt32Size);
  // We have to do two 32-bit stores here because
  // 1) tagged part modifications must be atomic to be properly synchronized
  //    with the concurrent marker.
  // 2) atomicity of full pointer store is not guaranteed for embedder slots
  //    since the address of the slot may not be kSystemPointerSize aligned
  //    (only kTaggedSize alignment is guaranteed).
  // TODO(ishell, v8:8875): revisit this once the allocation alignment
  // inconsistency is fixed.
  Address lo = static_cast<intptr_t>(static_cast<int32_t>(value));
  ObjectSlot(address() + kTaggedPayloadOffset).Relaxed_Store(Smi(lo));
  Address hi = value >> 32;
  ObjectSlot(address() + kRawPayloadOffset).Relaxed_Store(Object(hi));
#else
  ObjectSlot(address() + kTaggedPayloadOffset).Relaxed_Store(Smi(value));
#endif
}
```

注意到`EmbedderDataSlot::gc_safe_store`这个函数在开关指针压缩特性的时候有不同的实现，看起来在没开指针压缩的时候是用8个字节，开了指针压缩用4个字节

# Puer UStruct的包装大小

Puerts不同的对象有不同的包装处理，对于`UStruct`来说，其包装成本是4个额外的Internal Field。

```cpp
v8::Local<v8::FunctionTemplate> FStructWrapper::ToFunctionTemplate(v8::Isolate* Isolate, v8::FunctionCallback Construtor)
{
    auto ClassDefinition = FindClassByType(Struct.Get());
    bool IsReuseTemplate = false;
    ...
    auto Result = v8::FunctionTemplate::New(
        Isolate, Construtor, v8::External::New(Isolate, this));   
    Result->InstanceTemplate()->SetInternalFieldCount(4);
}


void FJsEnvImpl::BindStruct(
    FScriptStructWrapper* ScriptStructWrapper, void* Ptr, v8::Local<v8::Object> JSObject, bool PassByPointer)
{
    DataTransfer::SetPointer(MainIsolate, JSObject, Ptr, 0);
    DataTransfer::SetPointer(
        MainIsolate, JSObject, static_cast<UScriptStruct*>(ScriptStructWrapper->Struct.Get()), 1);    // add type info
}        
```

可以看到对一个UStruct:
4个InternalField分别存的是
- Object自己的Ptr(占据0-1 两个InternalField)
- UClass Ptr(占据2-3 两个InternalField)

实际上从V8的Heap Snapshopt抓出来来看，v8的对象应该还有额外的数据存储开销，一个UE.Vector 占56字节，原始指针8字节，包装成本48字节
加上UE.Vector实际在堆上分配了24字节(3个double)，总共80字节。
![edit-ad9749ebfb5245b784bdbbb961c2f727-2024-11-24-14-13-11](https://img.blurredcode.com/img/edit-ad9749ebfb5245b784bdbbb961c2f727-2024-11-24-14-13-11.png?x-oss-process=style/compress)



把`InternalField`改到6以后，大小从56变到了72字节，1个internalfield占据8字节。


![edit-ad9749ebfb5245b784bdbbb961c2f727-2024-11-24-14-13-33](https://img.blurredcode.com/img/edit-ad9749ebfb5245b784bdbbb961c2f727-2024-11-24-14-13-33.png?x-oss-process=style/compress)


# 为什么Puerts一个指针要拆成两个InternalField存储

```cpp
    FORCEINLINE static void SetPointer(v8::Isolate* Isolate, v8::Local<v8::Object> Object, const void* Ptr, int Index)
    {
        // Object->SetInternalField(Index, v8::External::New(Isolate, Ptr));
        // Object->SetAlignedPointerInInternalField(Index, Ptr);
        UPTRINT High;
        UPTRINT Low;
        SplitAddressToHighPartOfTwo(Ptr, High, Low);
        Object->SetAlignedPointerInInternalField(Index * 2, reinterpret_cast<void*>(High));
        Object->SetAlignedPointerInInternalField(Index * 2 + 1, reinterpret_cast<void*>(Low));
    }
```

最早读这段代码也比较疑惑，当时还在Puerts的讨论区问了一下。

> [[Question] 为什么SetPointer需要把指针拆成高4位和低4位 #1558](https://github.com/Tencent/puerts/discussions/1558)

当时车神给我解释了下，也没听懂。
后来自己构造了一个case弄懂了

```cpp

struct Foo
{
    char a; // zero offset
    char b; // 1 offset
}

Puerts.RegisterClass<Foo>()....
```

考虑以上Foo结构体的b字段，假设`Foo`的某个对象在8字节对齐的某个地址上，那么b字段将不能满足 **v8的`InternalField`存储的指针必须满足2字节对齐** 的要求。

> SetAlignedPointerInInternalField(): Sets a 2-byte-aligned native pointer in an internal field. To retrieve such a field, GetAlignedPointerFromInternalField must be used, everything else leads to undefined behavior. 