---
layout: post
title: "X11、Wayland、Mir"
tag: linux
category: 计算机
---



X11/Wayland/Mir 是 Linux 下的窗口系统，而 GNOME/KDE 是窗口管理器，也称作桌面环境，当程序想创建一个窗口时调用的是 GNOME/KDE 的接口，然后 GNOME/KDE 会再调用 X11/Wayland/Mir 在显卡上把窗口画出来。因此窗口系统更底层，且直接和显卡打交道。当然两者的界限也不是固定不变的，毕竟只是个软件的抽象层次的问题，底层干得越多，上层就可以干得越少，底层干不了的，上层就得自己来。这也是 Daniel Stone 说的，因为 X 越来越不给力，GNOME 已经快变成 X 了。

X Window System，简称 X11 或 X，由 MIT 在 1984 年开发，最后的主版本是 11。它跟 Wayland 的主要区别如下图：

![wayland-and-x11](http://omy6w6iwk.bkt.clouddn.com/wayland-and-x11-1.png)

在 X 中， XServer 负责跟 XClient 打交道，再将绘制需求发给 Compositor。Compositor 类似一个排版工，会把所有的图层合并为最终显示结果，再通知 XServer 绘制到显卡上。XServer 和 Compositor 是通过 IPC 通信的，在早期这不是大问题，因为输入设备单一，窗口也没有那么复杂，调用次数不多。但随着技术发展，现在的情况跟几十年前完全不同了，输入设备繁多，窗口也越来越炫，平板设备还能多点触控，IPC 通信增多后就影响刷新速度，X 到 2013 年就基本玩不转了，会出现闪动、卡死等问题。而且随着几十年的进化，X 的代码越来越臃肿，扩展越来越多，已经变得不可维护了，以前的 C/S 特色现在也基本废了，新的扩展根本不支持，所以再维护下去还不如开发一个全新的，这就有了 Wayland。

Wayland 实际就是将 Server 和 Compositor 合并了，减少了中间人，效率自然高。另外，X 还有一个缺陷是 Compositor 不能和输入设备 (`evdev`) 打交道，所以一个设备输入进来后 Server 可能根本不知道操作的是哪个窗口，因为排版的过程是 Compositor 做的，自己手上只有一个最终的平面图而已，是没有窗口间的逻辑关系的。而 Wayland 一手处理输入，一手又掌握排版，自然能灵活处理。

Wayland 兼容 X11，XServer 把 Wayland Compositor 当做自己的 Compositor 即可。

![wayland-and-x11](http://omy6w6iwk.bkt.clouddn.com/wayland-and-x11-2.png)

Mir 和 Wayland 类似，是 Canonical 公司开发的，用于自己的 Ubuntu 系统。而 Wayland 是 MIT License。

详情见 Daniel Stone 的视频：[The Real Story Behind Wayland and X - Daniel Stone](https://www.youtube.com/watch?v=RIctzAQOe44) 。

