---
title: "VirtualBox with Windows 11"
description: "How to get Windows 11 working on VirtualBox to run Windows-only applications"
date: "2023-07-13"
categories:
    - others
image: VirtualBox.svg
toc: true
toc-depth: 4
---

## Introduction

You might have probably guessed by my previous posts that I run Linux. In fact, I run exclusively Linux machine since 2010, my last Windows experience was Windows Vista and I don't like to remember that very well.

Unfortunately, even with the all powerful Wine, there are some applications that cannot be run on Linux, such as PowerBI. And since I am looking around what kind of industry skills one needs to have, PowerBI appeared quite a lot.

Initially, I tried to install it on Wine, but there is quite a lot of issues with the newest dotnet (`.NET`) libraries, the newer one just plainly do not work with any version of Wine and there is a long-standing bug. S

## VirtualBox

Last time we were talking about [containers](containers.md), specifically Podman and Docker. These are type of software virtualizations. They also have a bigger cousin called Virtual Machine, which allows for virtualization of a whole OS. If you are confused a bit like me, [this](https://stackoverflow.com/a/16048358/4868692) answer seems to explain the differences quite well.

Virtual Machines are also divided into multiple types, the main distinction is mainly between type 1 and type 2 Hypervisors. Type 1 runs virtual machine directly on hardware, while type 2 runs on the host OS layer. But let's top here.

When I was looking around for a good VM, [VirtualBox](https://www.virtualbox.org/) jumped at me. While it is from Oracle, it is free, open-source, and seems to be used quite a bit.

Another thing that jumped at me is that nowadays you can [download Windows for free](https://www.microsoft.com/software-download/windows11). Props to MS for this, looks like the times are getting better and barriers that hinders one to use software of their choice are falling.

### Not this way: VirtualBox 7.0 

As any reasonable person, first thing I did was download the latest version of VirtualBox and installed Windows 11.

The installation was breeze, and there wasn't anything that seemed wrong. However, issues started to pop and when I tried to run any app, I got reboots, freezing, and various graphical issues. Especially after I installed an extension that allows you to copy paste from one system to another or just drag and drop files, the operation system barely worked and nothing seemed to help.

Internet suggested, that there seems to be some incompatibility between VB 7.0 and Win11, and that the previous version of VB works fine.

### This way to go: VirtualBox 6.1

The first thing that surprised me is that installation process on the 6.1 was quite different than on the 7.0. For some reason, Windows 11 behaved differently already in this stage. The whole installation process was much more involved, notably it required MS account, which was not required on 7.0. Fortunately, in the end, everything worked perfectly, I could run and install PowerBI, and I was even able to test some Windows-only games that do not work on Wine.

So to stop me blaberring, are some tips and instructions for you if you are installing VM for the first time:

#### Use SSD and not HDD

During my first attempt, I used HDD, but this was a clear mistake. The time it takes to load VM is equivalent to time it takes to load OS on HDD, which means you will wait ages and the system will be quite slower. So just use SSD, the time it will take to load the whole OS won't be much different from the time it takes you to start browser.

#### Not enough resources bug

While on VB 7.0 the installation goes smoothly, stuff gets a bit rough on VM 6.1. At the start of the installation, the Windows will proclaim that it doesn't have enough resources, although 4GB ram and 64 GB space should be enough. You need to do some regedit:

* On the installation screen, before you click `INSTALL NOW`, press **SHIFT+F10** to start terminal
* type `regedit` to start Registry Editor
* Navigate to `HKEY_LOCAL_MACHINE\SYSTEM\Setup`
* Right-click on `Setup` and select `New => Key` with name `LabConfig`
* Right-click on `LabConfig`, select `New => DWORD (32-bit)` and create a value named `BypassTPMCheck`
* Set `BypassTPMCheck` value to `"1"`
* In the same way (`New => DWORD (32-bit)`), create `BypassRAMCheck` and `BypassSecureBootCheck` with values `"1"`
* Close the Registry Editor and close the terminal (e.g., type `exit`)
* Click `INSTALL NOW` and proceed with installation

#### Installing extension pack

When you are running Windows, click on `Devices` in the VM menu, and then `Insert Guest Additions CD Image`, which will download the image from the VirtualBox website.

Note that you _need_ then access the loaded image and install it BOTH on your host OS machine, and on the virtualized one. Only then you can enable the shared clipboard and drag-and-drop features. These should work after reboot.

## Summary

And you are there, you should be able to run PowerBI, or any Windows-only SW, on your Linux machine like I did. Hope some of this was helpful for you. See you in the industry!
