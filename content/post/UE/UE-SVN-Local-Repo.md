
---
title: "Deploy A Local SVN Repo For Unreal Source Control"
date: 2023-08-02T23:06:12+08:00
draft: false
categories: [ "UE"]
isCJKLanguage: true
slug: "4cec92aa"
toc: true 
mermaid: false
fancybox: false
# latex support
# katex: true
# markup: mmark
# mmarktoc: false 
UEVersion: 5.2.0
---


**Perforce** is the first class source control system for Unreal Engine, Epic itself, along with many other game studios, including my current studio, are using it.
Epic develops an excellent tool to work with Perforce [`UnrealGameSync`](https://docs.unrealengine.com/4.26/en-US/ProductionPipelines/DeployingTheEngine/UnrealGameSync/).

Although perforce is the best choice for Unreal Engine, it's not free, and it's not easy to setup a local perforce server for personal use.

It's common for developers, and also artists, to create some **Local** and **small** repos for prototyping ,bug testing and learning.
For example, developers may have a [`Lyra Sample`](https://docs.unrealengine.com/5.1/en-US/lyra-sample-game-in-unreal-engine/) as a playground and also a learning resource on their disks.
Personally, I may change code/assets from such testing project, to observe what may happen, but lately revert these experimental change.
Without source control, it's impossible to record what I changed and revert.


# Taking Subversion (SVN) as Source Control

To be honest, I'm not a fan of SVN due to my unfavorable experience about its fragility.
I once worked on a fairly large project(about 600GB) and SVN complained from time to time.
But `Git` is an even bigger disaster for game projects, even with `git-lfs`.

Back to the point, Epic has a documentation about how to deploy a local SVN repo, and associate it with Unreal Editor.

> Reference：[Using SVN as Source Control for Unreal Engine | Unreal Engine 5.2 Documentation](https://docs.unrealengine.com/5.2/en-US/using-svn-as-source-control-for-unreal-engine/)

It's a good documentation, but it's just a little bit outdated and is based on the `VisualSVN`.
I have tested tortoiseSVN and it was OK, and I need to make some additions to make it work.


## Error: Expected FS format error

> 参考：[SVN source control error within UE4 - Development / Pipeline & Plugins - Epic Developer Community Forums](https://forums.unrealengine.com/t/svn-source-control-error-within-ue4/458469)

`svn: E160043: Expected FS format between '1' and  '7'; found format '8'  `

This error is because unreal doesn't use system installed subversion, but it downloads an old version(`svn, version 1.9.5 (r1770682)`) during `Setup.bat`.
See `<EnginePath>/Binaries/ThirdParty/svn/Win64/svn.exe`.

To fix this, we need to create our local repo in using commandline tool with additional flag:

```
svnadmin create --compatible-version 1.9 --fs-type fsfs <repoName>
```

## Error: Could not be successfully checked out assets

> 参考：[How to solve “could not be successfully checked out” with SVN and Unreal Engine 4 Editor - IT World](https://www.mirabulus.com/it/blog/2020/08/14/how-to-solve-%E2%80%9Ccould-not-be-successfully-checked-out%E2%80%9D-with-svn-and-unreal-engine-4-editor)

It's an another common pitfall for local repo.
For a newly created local SVN repo, there is no security check and no username and password are needed.
In Unreal Editor Source Control plugin, inputting repo url without specifying username and password may seems work(Unreal Editor won't complains).
But when you try to check out files, it will fail.

This is because Unreal Editor will execute `SVN Lock` command with the current system user name.
And after files are locked, the editor plugin will check if the files is locked by ourselves, but oops, we haven't specified any username and password, so the plugin fails the check and thinks the files are locked by others.

To fix it, just specify a username (password is not needed) in the below window.

![UE-SVN-Local-Repo-2023-08-02-23-41-00](https://img.blurredcode.com/img/UE-SVN-Local-Repo-2023-08-02-23-41-00.jpg?x-oss-process=style/compress)







