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

```
import com.google.common.collect.Lists;

import java.util.List;
import java.util.concurrent.CompletableFuture;
import java.util.concurrent.ExecutionException;

public class FutureTest {

    public static void main(String[] args) throws ExecutionException, InterruptedException {
        new FutureTest().run();
    }

    private void run() throws ExecutionException, InterruptedException {
        List<CompletableFuture> futures = Lists.newArrayList();
        for (int i = 0; i < 1000; i++) {
            CompletableFuture future = new CompletableFuture();
            CompletableFuture f = CompletableFuture.runAsync(this::read);
            f.whenComplete((r, e) -> future.complete(r));
            futures.add(future);
        }
        for (CompletableFuture future : futures) {
            future.get();
        }
    }

    private void read() {
        try {
            Thread.sleep(100);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
    }
}
```



```
import com.google.common.base.Stopwatch;
import org.junit.Assert;
import org.junit.Test;

import java.util.concurrent.TimeUnit;

public class AvailableProcessorsTest {

    @Test
    public void test() {
        Stopwatch sw = Stopwatch.createStarted();
        for (int i = 0; i < 1000000; i++) {
            Runtime.getRuntime().availableProcessors();
        }
        Assert.assertTrue(sw.elapsed(TimeUnit.SECONDS) > 10);
    }
}
```