---
layout: post
title: Memory Barrier
description: memory barrier
category: blog
---

software barrier:
`asm volatile ("" : : : "memory");` 被广泛的应用在Linux kernel中，用作barrier.
那么它的作用是什么呢? 它是编译器调度barrier，在barrier之前的load/store操作不可以
被放在barrier之后，在barrier之后的load/store操作不可以放在barrier之前.

https://gcc.gnu.org/ml/gcc-help/2011-04/msg00166.html
```
The definition of "memory barrier" is ambiguous when looking at code
written in a high-level language.

The statement "asm volatile ("" : : : "memory");" is a compiler
scheduling barrier for all expressions that load from or store values to
memory.  That means something like a pointer dereference, an array
index, or an access to a volatile variable.  It may or may not include a
reference to a local variable, as a local variable need not be in
memory.

This kind of compiler scheduling barrier can be used in conjunction with
a hardware memory barrier.  The compiler doesn't know that a hardware
memory barrier is special, and it will happily move memory access
instructions across the hardware barrier.  Therefore, if you want to use
a hardware memory barrier in compiled code, you must use it along with a
compiler scheduling barrier.

On the other hand a compiler scheduling barrier can be useful even
without a hardware memory barrier.  For example, in a coroutine based
system with multiple light-weight threads running on a single processor,
you need a compiler scheduling barrier, but you do not need a hardware
memory barrier.

gcc will generate a hardware memory barrier if you use the
__sync_synchronize builtin function.  That function acts as both a
hardware memory barrier and a compiler scheduling barrier.k
```

有人问下面的例子中，x会不会被优化，导致没有访存操作，然后memory barrier就无效了?
```
static int x;
void test(void) {
	x = 1;
	asm volatile ("" : : : "memory");
	x = 2;
}
```
见如下回答，通过做实验发现写x的操作不会被优化成寄存器赋值，依旧是store操作.
https://gcc.gnu.org/ml/gcc-help/2011-04/msg00183.html
```
I don't think either assignment can be removed, at least not when
the program is compiled with threads enabled.

Even though x is local to this translation unit it is not local to
this thread.  The compiler must respect memory barriers in this
case: if it did not, POSIX threads support would not work.  There
was a fantastically long thread about this
"Optimization of conditional access to globals: thread-unsafe?" in
Oct 2007.
```
