
---
title: "Mesa | VK_EXT_PRIVATE_DATA的实现"
date: 2024-09-02T20:54:33+08:00
draft: false
categories: [ "Mesa"]
isCJKLanguage: true
slug: "7cf58aab"
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


这个扩展允许在任意的VK_OBJECT上绑定若干个8字节的数据。
主要用法是:

- 通过vkCreatePrivateDataSlotEXT创建一个slot,这个slot内部结构是不透明的，但是实际上里面就是一个`index`(应用层不可见)
- 这个slot可以被作为索引访问修改一个VK_OBJECT上的PRIVATE_DATA

# 问题1: 为什么Index要做成不透明的

这个Index实际上是存放在vkDevice上的一个成员变量(驱动层), 每次创建一个slot,这个index就会自增1。
这样可以避免不同的Layer申请到同一个Index。
每个Layer和应用每次调用`vkCreatePrivateDataSlotEXT`都会创建一个自增的index的slot。

# 问题2: 数据存放在哪， 数据结构是什么

数据实际存放在所有vulkan object的基类 `vk_object`上，数据结构是sparse array。
通过index可以稀疏的创建和删除数据。

# 问题3：为什么API需要传入ObjectType

既然PrivateData在所有的VK_OBJECT上，理论上是可以在任何的VK_OBJECT上绑定数据，为什么API需要传入ObjectType

```cpp
// Provided by VK_VERSION_1_3
VkResult vkSetPrivateData(
    VkDevice                                    device,
    VkObjectType                                objectType, // ???Why
    uint64_t                                    objectHandle,
    VkPrivateDataSlot                           privateDataSlot,
    uint64_t                                    data);
```

因为Swapchain似乎比较特殊，具体哪里特殊暂时还不理解，至少对swapchain有额外的处理。

```cpp
   if (objectType == VK_OBJECT_TYPE_SURFACE_KHR) {
   ...
      mtx_lock(&device->swapchain_private_mtx);
      VkResult result = get_swapchain_private_data_locked(device, objectHandle,
                                                          slot, private_data);
      mtx_unlock(&device->swapchain_private_mtx);
      return result;
   }

   struct vk_object_base *obj =
      vk_object_base_from_u64_handle(objectHandle, objectType);
```

# 核心接口

核心的几个内部实现就在这里了。

## VkPrivateDataSlot的数据结构
```c
struct vk_private_data_slot {
   struct vk_object_base base;
   uint32_t index; // 一个Slot实际上只是一个VkObject + uint32_Index
};

VK_DEFINE_NONDISP_HANDLE_CASTS(vk_private_data_slot, base,
                               VkPrivateDataSlot,
                               VK_OBJECT_TYPE_PRIVATE_DATA_SLOT);
```
## Slot的创建 

```cpp
VkResult
vk_private_data_slot_create(struct vk_device *device,
                            const VkPrivateDataSlotCreateInfo* pCreateInfo,
                            const VkAllocationCallbacks* pAllocator,
                            VkPrivateDataSlot* pPrivateDataSlot)
                            {
    vk_object_base_init(device, &slot->base,
                       VK_OBJECT_TYPE_PRIVATE_DATA_SLOT);
    // 可以看到index = device->private_data_next_index++ 
    slot->index = p_atomic_inc_return(&device->private_data_next_index);
}

```

