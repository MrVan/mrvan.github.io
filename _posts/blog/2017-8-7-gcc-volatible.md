---
layout: post
title: gcc volatile
description: gcc volatile
category: blog
---

C语言中有volatile对象，他们通常用来通过指针来访问硬件或用做进程间通信. C标准
只是说对于volatile对象的访问，编译器应该不做优化，但是它是implementation
defined as to what constitutes a volatile object.

对于non-volatile对象的访问不需要和volatile access保持顺序(ordered),因此
Volatile 不可以用来做memory barrier来使编译器编译程序后写的顺序和程序员编写
程序时的顺序一致。看下面的例子

```c
int *ptr = something;
volatile int vobj;
*ptr = something;
vobj = 1;
```

编译器不能保证更新`*ptr`和vobj一定是顺序的，除非ptr和vobj是aliased，如果需要
他们保持顺序，需要加`asm volatile ("" : : : "memory");`
