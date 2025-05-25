---
layout: post
title: Patching .NET applications
date: 2025-05-25
---
# Introduction
In the modern age, Windows reigns amongst the most used systems. Such is this fact that also many developers dedicate exclusively to the .NET framework.

When debugging, researching and reversing .NET apps, we will find ourselves often wondering how to speed up some processes, skip product key checks, fix bugs on the fly or even adding custom features to closed-source software.

In this post we will navigate through the basics of patching/modifying .NET applications (based in C#) to accomplish some of these tasks.
## Disclaimer
This is only for research and educational purposes. This does not aim in any way to encourage piracy or any other malicious intent against any kind of software.
## Case of study
To demonstrate how to do this, we will be dissecting a deprecated, old, non-working version of a cracking tool found in obscure forums.
# Requirements
- [ILSpy](https://github.com/icsharpcode/ILSpy)
- [dnSpyEx](https://github.com/dnSpyEx/dnSpy) (fork of dnSpy)
- [HxD](https://mh-nexus.de/en/hxd/)
- Any debugger, such as [x64dbg](https://x64dbg.com/) or [Cheat Engine](https://cheatengine.org/)
