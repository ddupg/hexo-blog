---
layout:     post
title:      "「记」Jdk8 CompletableFuture在高并发环境下的性能问题"
subtitle:   "Jdk8 CompletableFuture在高并发环境下，性能受Runtime.availableProcessors()影响"
date:       2019-10-11 19:30:00
author:     "Hardbird"
header-img: ""
catalog: true
tags:
    - Jdk
    - Java
---

[JDK-8227018](https://bugs.openjdk.java.net/browse/JDK-8227018)

![火焰图](../img/CompletableFuture/CompletableFuture-in-jdk8-traces.svg)