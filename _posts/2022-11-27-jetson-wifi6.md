---
layout:     post
title:      "WiFi 6 on Jetson Xavier NX"
subtitle:   ""
date:       2022-11-27 22:30:00
tags:
    - Misc
typora-root-url: ../
published: True
---

**TL;DR:** pass the USB device in a VM with higher version Linux kernel, but need to remove several lines in the kernel source and recompile the kernel to make KVM work on Jetson (may be unnecessary with later kernel updates).

I want to use a WiFi 6 USB adapter ([CF-953AX](http://www.comfast.cn/index.php?m=content&c=index&a=show&catid=13&id=149)) on Jetson Xavier NX.
This is quite tricky because, the kernel driver (mt7921u) is available only after Linux 5.19, while the newest Jetson SDK comes with Linux 5.10.

A straight thought would be to upgrade the Linux kernel on Jetson board.
However, that would require a lot of customization and kernel patching, which is tedious and error-prone.
Then I tried to look for backport of mt7921u driver, but no result (please let me know if there is!).

Then one idea comes to my mind: why not just **drive the USB device in a VM, and bridge the network to the host**?
There will be no risk of messing up the host kernel, and I will be free to use newest Linux kernel, just with a little performance loss, which is definitely worthwhile.

Then I downloaded Ubuntu 22.10 server version for arm64, which comes with Linux 5.19.
To ease the setup, I used virt-manager.
I believe there is no need to remind the readers about how to setup the VM with USB passthrough, just tweak in the virt-manager UI, one can easily find these settings.

Unfortunately, a trial booting gives error message about PMU:

![kvm-pmu-error](/img/posts/kvm-pmu-error.png)

Obviously, Jetpack 5 kernel comes with KVM support (before Jetpack 5, you need to manually add the support, check [this](https://forums.developer.nvidia.com/t/guide-to-enable-kvm-on-the-xavier/119777)).
It is related to PMU.
Some googling finds [this](https://discuss.linuxcontainers.org/t/vms-do-not-start-on-lxd-4-10-4-11-on-aarch64-with-kernel-5-10/10227/7).

To be brief, there are some commits about PMU in Linux 5.11, some of them are already ported back to 5.10 while others are not.
Just so bad luck…
Well, a simple skim on what has not been ported stops me from trying backporting it.
Let's just do it in another direction, namely remove [the backported commit](https://patchwork.kernel.org/project/linux-arm-kernel/patch/20210107112101.2297944-2-maz@kernel.org/):

![kvm-patch](/img/posts/kvm-patch1.png)

As the commit message implies, it is not critical. After manually removing this patch, I compiled and replaced the Linux kernel (see [this](https://github.com/ShengliangD/shengliangd.github.io.git)), and reboot.
Then the VM starts, and the USB WiFi adapter is recognized in the VM.
Unfortunately, a quick test with iperf3 only got around 18Mbps speed, which is much slower than [others' test](https://github.com/morrownr/USB-WiFi/discussions/88)(~500Mbps).
I will look into it and share the progress later.
Anyway, the WiFi adapter is working in the VM, and I believe there is no need to mention how to set up routing rules to bridge the network between the host and the VM.
