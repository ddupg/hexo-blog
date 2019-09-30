---
layout:     post
title:      "「记」Java类初始化顺序"
subtitle:   "子类复写父类钩子方法与类初始化顺序的一个注意点"
date:       2019-09-30 10:00:00
author:     "Hardbird"
header-img: "img/HBaseArchitecture/home-bg-o.jpg"
catalog: true
tags:
    - HBase
    - translation
---

## bug现象

```
public class Father {

    public Father() {
        init();
    }

    protected void init() {
    }

}
```

```
public class Child extends Father {

    int a = 0;

    public Child() {
        super();
    }

    @Override
    protected void init() {
        super.init();
        a = 1;
    }
}
```

```
import org.junit.Test;

import static org.junit.Assert.*;

public class ChildTest {

    @Test
    public void testConstructor() {
        Child child = new Child();
        assertEquals(0, child.a);
    }
}
```