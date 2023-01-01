---
layout:     post
title:      "WiFi 6 on Jetson Xavier NX"
subtitle:   ""
date:       2022-12-30 10:30:00
tags:
    - Tech
typora-root-url: ../
published: True
---

I want to use a WiFi 6 USB adapter ([CF-953AX](http://www.comfast.cn/index.php?m=content&c=index&a=show&catid=13&id=149)) on Jetson Xavier NX.
This is quite tricky because the kernel driver (mt7921u) is available only after Linux 5.19, while the newest Jetson SDK comes with `5.10.104-tegra`.
A straight thought would be to upgrade the Linux kernel on Jetson board.
However, that would require a lot of customization and kernel patching, which is tedious and error-prone.
Then I tried to look for backport of mt7921u driver, but no result.

Then one idea comes to my mind: why not just **drive the USB device in a VM, and bridge the network to the host**?
There will be no risk of messing up the host kernel, and I will be free to use newest Linux kernel.
It is definitely worth a try, but finally I could only get \<20Mbps speed, and haven't found the root cause yet.
In the end I decide to backport the mt7921u driver from Linux 5.19, described in my [next blog]().
Check out my [repo](https://github.com/ShengliangD/mt76-backport.git) if you need it.

Although the VM solution is not perfect, it is still an interesting idea, and the steps are shared in this post.

# Setup the VM

I downloaded Ubuntu 22.10 server version for arm64, which comes with Linux 5.19.
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
Let's first make the host able to access the WiFi, then debug the speed issue.

# Setup Routing

Now, the VM is up and running with a USB WiFi adapter.
But how do we access the WiFi from the host?

First, make sure the host and the VM can access each other.
This can be done by setting up a bridged network.
Then, set up routing on the host so that access to WiFi ips from the host goes to the VM.
Finally, set up NAT on the VM with the following commands:

```bash
# enable ip forwarding
sysctl -w net.ipv4.ip_forward=1
iptables --policy FORWARD ACCEPT
# set up NAT
iptables -t nat -A POSTROUTING -o <vm_wnic> -j MASQUERADE
```

However, in this way, the host hides behind NAT and cannot be accessed by devices in WiFi.
We actually would like to make the VM just like a driver, or to be more specific, make the WiFi IP of the VM behave like an IP of the host.
First of all, ip forwarding in the VM is still necessary:
```bash
sysctl -w net.ipv4.ip_forward=1
iptables --policy FORWARD ACCEPT
```

Then, on the VM, all output packets to WiFi subnet should rewrite its source IP to the VM's WiFi IP, such that the remote host response to the VM's WiFi IP instead of the host's IP:
```bash
iptables -t nat -A POSTROUTING -o <vm_wnic> -j SNAT --to-source <vm_wifi_ip>
```

Still, on the VM, all input packets from WiFi should be rewritten to the host's IP:
```bash
iptables -t nat -A PREROUTING -i <vm_wnic> -j DNAT --to-destination <host_ip>
```

Now the WiFi IP of the VM behaves like an IP of the host.
