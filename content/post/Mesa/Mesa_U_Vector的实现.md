
---
title: "Mesa | u_vector的实现"
date: 2024-09-11T00:05:57+08:00
draft: false
categories: [ "Mesa"]
isCJKLanguage: true
slug: "07cd164d"
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

可以看u_vector.c里的注释:

```
 * A dynamically growable, circular buffer.  Elements are added at head and
 * removed from tail. head and tail are free-running uint32_t indices and we
 * only compute the modulo with size when accessing the array.  This way,
 * number of bytes in the queue is always head - tail, even in case of
 * wraparound.
 */
```
主要特点:
- 可以动态扩容
- ring buffer


实现位于 ` src/util/u_vector.c`
 
 和`std::vector`的异同点 ：
 
 相同:
 - 尾部追加时可扩容，扩容系数2
 
 不同:
 
 - 不保证数据从0偏移量开始，head指针有可能有偏移量
 - 不允许随机访问，只能头尾访问.
 - 数据在内存上可能有环回，不连续，不能直接memcpy




```cpp

/* 

   |1|2|3|4| | | | 
   ^       ^
  tail    head

*/
struct u_vector {
   uint32_t head; 
   uint32_t tail;
   uint32_t element_size;
   uint32_t size;
   void *data;
};
```

head和tail的关系:
- 允许head和tail的index大于size(循环数组)，但是head和tail的差值不能大于size，只在必要的时候取余。不然的话，如果head在增长的时候取余，将会破坏head > tail这个不变量，也会导致(head - tail)不准确
- head和tail的差值表示有效元素的个数

```cpp
static inline int
u_vector_length(struct u_vector *queue)
{
   return (queue->head - queue->tail) / queue->element_size;
}
```

# 取头尾元素 head() / tail()

注意head指向的元素是invalid的元素(类似于std::vector::end)，当要取head的元素时，需要回退一个元素

```c
static inline void *
u_vector_head(struct u_vector *vector)
{
   assert(vector->tail < vector->head);
   return (void *)((char *)vector->data +
                   ((vector->head - vector->element_size) &  // 回退一个元素
                    (vector->size - 1)));  //取模
}
```


# 扩容策略： 2倍扩容

- trivial情况: 一直add到容量不够，这个时候直接扩容
- 不平凡的情况: 一边有add一边有remove，此时tail和head index都会移动，扩容的时候会引起两次拷贝。并且可能会形成两种情况:
(主要是根据tail的位置会影响扩容后是否有环回)
1. 之前有环回(循环数组)，扩容后由于容量增加，可能会变成线性数组
2. 扩容后仍然是环回数组

```c
/*

|4| 1| 2 | 3 |
head = 5, tail = 1, original size = 4, new size = 8
split = 4
src_tail = 1, dst_tail = 1

第一种情况，扩容后变成线性数组
memcpy(data + 1, vector_data + 1, 3)
memcpy(data + 5, vector_data, 1)

| | 1| 2 | 3 |4| | | |

第二种情况，扩容后仍然是环回数组
head = 9, tail = 5, orig size = 4, new size = 8
split = 8
src_tail = 1, dst_tail = 5
|4 | |  |  ||1 |2 |3 |
memcpy(data + 5, vector_data + 1, 3)
memcpy(data + 0, vector_data, 1)

*/
```



```c
void *
u_vector_add(struct u_vector *vector)
{
   uint32_t offset, size, split, src_tail, dst_tail;
   void *data;

   if (vector->head - vector->tail == vector->size) {
      size = vector->size * 2;
      data = malloc(size);
      if (data == NULL)
         return NULL;
      src_tail = vector->tail & (vector->size - 1);
      dst_tail = vector->tail & (size - 1);
      if (src_tail == 0) {
         /* Since we know that the vector is full, this means that it's
          * linear from start to end so we can do one copy.
          */
         memcpy((char *)data + dst_tail, vector->data, vector->size);
      } else {
         /* In this case, the vector is split into two pieces and we have
          * to do two copies.  We have to be careful to make sure each
          * piece goes to the right locations.  Thanks to the change in
          * size, it may or may not still wrap around.
   
          */
         split = align(vector->tail, vector->size);
         assert(vector->tail <= split && split < vector->head);
         memcpy((char *)data + dst_tail, (char *)vector->data + src_tail,
                split - vector->tail);
         memcpy((char *)data + (split & (size - 1)), vector->data,
                vector->head - split);
      }
      free(vector->data);
      vector->data = data;
      vector->size = size;
   }

   assert(vector->head - vector->tail < vector->size);

   offset = vector->head & (vector->size - 1); // 在 size == 2^n 的情况下，等价于 vector->head % vector->size
   vector->head += vector->element_size;

   return (char *)vector->data + offset;
}
```