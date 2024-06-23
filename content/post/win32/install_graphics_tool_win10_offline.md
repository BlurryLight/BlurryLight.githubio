
---
title: "Fix DirectX Graphics Tools On Win10 Failed To Install"
date: 2023-07-12T22:43:08+08:00
draft: false
categories: [ "win32"]
isCJKLanguage: true
slug: "190d19e2"
toc: true 
mermaid: false
fancybox: false
# latex support
# katex: true
# markup: mmark
# mmarktoc: false 
---


中文关键词: [修复DirectX图形工具无法安装的问题](/2023/07/190d19e2/)

# Graphics Tool

![edit-18e59d19bfd94450a2e7399a106cc38b-2022-11-14-13-53-33](https://img.blurredcode.com/img/edit-18e59d19bfd94450a2e7399a106cc38b-2022-11-14-13-53-33.png?x-oss-process=style/compress)

`Graphics Tool` is one of the essential components of Graphics Programming on Windows.
During the installation of Visual Studio, the installer tries to automatically install `Graphics Tool`, but things are not always going well.
In complicated network environment, such as in a company that has a strict firewall, or in the Chinese mainland which is blocked by the GFW, it is somewhat difficult to install `Graphics Tools`.

# UseWUServer = 0

This method is suitable for those who have a [WSUS](https://www.techtarget.com/searchwindowsserver/definition/Windows-Server-Update-Services-WSUS) server in their company, and have access to the Internet.
I encountered the situation while working for Tencent.

The clients are all configured to use an internal WSUS server, and sadly, although it seems that my client can acquire the latest updates, it cannot successfully install any `Optional Features` from the `Settings`, including the `Graphics Tools`.

So, the way to solve it is to temporarily redirect our WSUS to the official M$ server and install what we need, and then  restore the configuration.


- Open `regedit`, go to `HKEY_LOCAL_MACHINE\SOFTWARE\Policies\Microsoft\Windows\WindowsUpdate\AU`, change the value of `UseWUServer` to 0, which disables the user-defined WSUS.
- Open Administrator cmd, call `net stop wuauserv && net start wuauserv` to restart `windows update` service.
- Install `Graphics Tools` from `Settings` -> `Optional Features`.
- reset `UseWUServer` to 0, restart the WU service.


# Offline installer

The above method requires access to the M$ server, which is not possible when working in a completely offline environment.
That's the case when my client is behind a firewall and only a few sites are whitelisted, and of course, Microsoft is not on the list.

I must point out that M$ doesn't provide any available solution for this case.
They boastfully stats that these `Optional Features` can be installed from [a seperate `Features On Demand` ISO](https://learn.microsoft.com/en-us/windows-hardware/manufacture/desktop/features-on-demand-v2--capabilities?view=windows-11), but no one(perhaps excluding Windows developers) knows where to download the ISO.
It seems the ISOs are only provided to some OEMs and not distributed publicly.

The only way I found is provided in this article 

> Reference：[How to install Windows 10 1809 “Features On Demand” Without Internet Access – vCloudInfo](https://www.vcloudinfo.com/2019/01/how-to-install-windows-10-1809-features.html)

In a nutshell, when installing optional features, Windows Update downloads the installer, installs it, and then automatically deletes the installer.
Therefore, we can quickly copy the installer out in a very short time window (about several seconds) between when it is fully downloaded and when it gets deleted.
Luckily, we can try infinite times by uninstalling and reinstalling the feature.

After trying about 5 times, I finally captured an complete `Graphics Tools` installer.
It was captured on a `Windows 10 22H2` system, but I confidently guess it is going to work across versions.
Anyway, the method mentioned before is universal.

1. Download the [Microsoft-OneCore-Graphics-Tools-Package~31bf3856ad364e35~amd64~~.cab](https://drive.google.com/file/d/1Y_rxOWz1ClqSCR4PHIGutN8fz-dHHzly/view?usp=drivesdk) to a favorite directory
2. At that directory, Open shell. Make sure your current directory is same with the `cab`
3. Employ DISM to install it locally. 

```
DISM /online /add-package /packagepath:Microsoft-OneCore-Graphics-Tools-Package~31bf3856ad364e35~amd64~~.cab`
```

Check the DebugLayer dlls are installed.

![install_graphics_tool_win10_offline-2023-07-12-23-26-58](https://img.blurredcode.com/img/install_graphics_tool_win10_offline-2023-07-12-23-26-58.png?x-oss-process=style/compress)
![install_graphics_tool_win10_offline-2023-07-12-23-28-05](https://img.blurredcode.com/img/install_graphics_tool_win10_offline-2023-07-12-23-28-05.png?x-oss-process=style/compress)





