
---
title: "KDE_Wayland_Nvidia登陆后黑屏的问题"
date: 2024-11-03T21:39:42+08:00
draft: false
categories: [ "Linux"]
isCJKLanguage: true
slug: "d02b0f52"
toc: false
mermaid: false
fancybox: false
blueprint: false
# latex support
# katex: true
# markup: mmark
# mmarktoc: false 
# UEVersion: 5.3.2 
---


这两天重新装了Manjaro KDE作为桌面。因为双显示器的缩放问题(x11只能设置全局缩放不能设置per monitor缩放)只能用wayland。 结果装了Nvidia的专有驱动以后登陆进去就黑屏。(其实Mesa的NVK倒也不是不能用，只是我起了虚幻以后发现nvk可能性能还是不太行，有点卡卡的)。

查了半天从manjaro的论坛里发现了一个类似症状的帖子:

https://forum.manjaro.org/t/cannot-use-wayland-with-kde-on-nvidia/163318/8


翻了一下有大佬给了解决方案

```
    No plasma interface with kernel 6.9 + Nvidia gpu + Wayland

    If you encouter a black screen with no inteface after login in, it’s probably a problem with simpledrm loading.

    To solve it add nvidia_drm.fbdev=1 to /etc/default/grub
    in the line begining with GRUB_CMDLINE_LINUX=" .
    Verify that you also have nvidia_drm.modeset=1 in the same line.
    Then exec sudo update-grub

    Also, verify that you have nvidia_drm in /etc/mkinitcpio.conf in the MODULES= or HOOKS= line.
    Exemple :

    MODULES=(nvidia nvidia_modeset nvidia_uvm nvidia_drm)

    If it was not present, then run sudo mkinitcpio -P after adding it.

```

1. 修改/etc/default/grub,在GRUB_CMDLINE_LINUX=""里添加 `nvidia_drm.modeset=1 nvidia_drm.fbdev=1`，运行sudo update-grub
2. 在/etc/mkinitcpio.conf里添加 `MODULES=(nvidia nvidia_modeset nvidia_uvm nvidia_drm)`，运行`sudo mkinitcpio -P`,然后重启
