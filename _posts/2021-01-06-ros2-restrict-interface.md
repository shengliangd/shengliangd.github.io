---
layout:     post
title:      "ROS2 网卡白名单"
subtitle:   ""
date:       2021-01-06 11:00:00
tags:
    - Misc
typora-root-url: ../
published: true
---

最近做实验用到 ROS2，需要限制通信时可用的网卡，解决方案不太好找，google 一番先大致确定了是要针对 DDS 设置（关于 ROS2 架构、DDS 等可参考[这里](https://index.ros.org/doc/ros2/Concepts/DDS-and-ROS-middleware-implementations/)），然后针对 ROS2 默认的 fastrtps 搜索。先是找到[这里](https://eprosima-fast-rtps.readthedocs.io/en/latest/advanced.html#whitelist-interfaces)，然而并没有效果。后来在官方文档另一个页面 [Advanced Functionalities](https://fast-rtps.docs.eprosima.com/en/v1.7.0/advanced.html) 下发现还需要 `<useBuiltinTransports>false</useBuiltinTransports>`。完整的配置文件如下，把 IP1、IP2 等换成相应网卡的 IP 即可：

```xml
<?xml version="1.0" encoding="UTF-8" ?>

<profiles>
        <transport_descriptors>
                <transport_descriptor>
                        <transport_id>my_transport</transport_id>
                        <type>UDPv4</type>
                        <interfaceWhiteList>
                                <address>IP1</address>
                                <address>IP2</address>
                        </interfaceWhiteList>
                </transport_descriptor>
        </transport_descriptors>
        <participant profile_name="participant_profile" is_default_profile="true">
                <rtps>
                        <userTransports>
                                <transport_id>my_transport</transport_id>
                        </userTransports>
                        <useBuiltinTransports>false</useBuiltinTransports>
                </rtps>
        </participant>
</profiles>
```
