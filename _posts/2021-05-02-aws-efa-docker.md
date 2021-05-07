---
layout:     post
title:      "Using AWS Elastic Fabric Adaptor in Container"
subtitle:   ""
date:       2021-05-02 22:00:00
tags:
    - Misc
typora-root-url: ../
published: false
---

Recently I need to run some experiment on AWS with two servers, GPU, RDMA.
p3dn.24xlarge is the best choice after comparison.

My first chance to use high-end instances on AWS... Some problems met.

My env was configured in Docker.
But inside docker, `ibv_devinfo`: 'warning: no userspace device-specific driver found for'.
it's clear that AWS uses its own userspace driver.
So we just need to find it and install in the container.

Some googling find aws efa installer (here)[https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/efa-start.html#efa-start-impi].
Although that tutorial is using yum, etc., things would be similar.
I downloaded, extracted, found debs.

So install using the script.
kernel not detected.
simplest way: install a kernel (but we won't use it because we are in container.)

Now kernel ok, but some packet overwrite problem.
My guess: aws conflict with official rdma related packages.
Remove them if you have. For me: `apt remove ibacm ibverbs-providers infiniband-diags libibmad-devel libibumad-devel librdmacm-utils librxe-1 libibumad libmlx5-1 libmlx4-1`

Then install using the script.
Check with `ibv_devinfo` see if things working.

## Some Notes about AWS EC2

Sometimes you find no GPU/EFA...

You can store AMI for later use.
