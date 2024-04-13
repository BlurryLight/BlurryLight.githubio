
---
title: "Puerts | 利用V8机制进行并行GC"
date: 2024-04-13T16:18:20+08:00
draft: false
categories: [ "JavaScript", "PuerTs", "UE"]
isCJKLanguage: true
slug: "f1685bcf"
toc: true
mermaid: false
fancybox: false
blueprint: false
# latex support
# katex: true
# markup: mmark
# mmarktoc: false 
---


这里讨论的东西其实是利用v8的机制，所以理论上nodejs的也能用上。

# Finalizer的性能问题和可靠性问题

在用脚本语言和宿主语言交互的时候，不可避免的会创建许多临时的js object对象，比如在ts里调用各种C++函数，返回的结果都必须包装为js object。

最早Puerts全部使用的是`v8::PersistentBase::SetWeak`的方法，这也是v8的hello world文档里讲的方法(https://v8.dev/docs/embed)。
它允许在C++里指向js object的Persistant handle并设置为WeakPtr，当这个WeakPtr是最后一个存在的reference的时候，v8会调用我们指定的callback来执行资源释放的操作。

它的问题主要是:
- v8不保证这个callback一定会被执行，有可能会有资源泄露(Finalizer问题，许多GC语言都会有这个问题，有的语言甚至允许在finalizer里重新复活这个对象)
- 执行的时候是在主线程，大量的小对象回收会阻塞主线程很长时间

# ArrayBuffer BackStoring


v8 8加入的新机制，允许创建一个`v8::arrayBuffer`的时候指定所需要的`BackStoring`,一个`BackStoring`代表一段raw内存，这段内存的声明周期会和这个`v8::arrayBuffer`绑定到一起，当这个`v8::arrayBuffer`被GC回收的时候，会执行用户指定的callback来释放这段内存。

这个方案的优势是:

- 从v8 8.3起，所有的array buffer的GC从串行变为并行GC，不会阻碍主线程，见[V8 release v8.3](https://v8.dev/blog/v8-release-83#faster-arraybuffer-tracking-in-the-garbage-collector)
- 代码写起来更舒服，不用考虑Persistant handle存到哪里的问题

# 结合两个方法

最早Puerts的实现是所有的对象都改成了由ArrayBuffer管理，但是后面发现了一个问题，就是虚幻的UObject/UStruct这种对象需要在主线程上保证线程安全，而ArrayBuffer的GC释放是在worker thread上，这就导致会在worker thread上触发U类的析构函数，这是不允许的。

再次思考:
- 通过`v8::arraybuffer`管理的内存，其GC释放是在worker thread上
- 通过`PersistentBase<T>::SetWeak`管理的内存，其GC释放是在main thread上

我们可以把这两种机制结合起来，把需要在主线程上释放的对象用`PersistentBase<T>::SetWeak`管理，其他的对象用`v8::arraybuffer`管理。
由于我们无法判断一个类的析构函数是否是线程安全的，只能做的保守一点，对于POD的类型，我们可以用`v8::arraybuffer`管理，对于其他类型，我们用`PersistentBase<T>::SetWeak`管理。

Puerts去年合进去了这个PR [Free POD UStruct on Worker Thread. Fix #1539 by BlurryLight · Pull Request #1576 · Tencent/puerts](https://github.com/Tencent/puerts/pull/1576).
