---
layout: post
title: asm volatile
description: what does asm volatile means?
category: blog
---
In one words, asm volatile to is protect code from being optimzed by compiler.

See: [gcc extend.texi](https://fossies.org/linux/gcc/gcc/doc/extend.texi)

The typical use of extended asm statements is to manipulate input values
to produce output values. However, your asm statements may also
produce side effects. If so, you may need to use the volatile qualifier
to disable certain optimizations.

```
GCC’s optimizers sometimes discard asm statements if they determine there
is no need for the output variables. Also, the optimizers may move code out
of loops if they believe that the code will always return the same result
 (i.e. none of its input values change between calls). Using the volatile
 qualifier disables these optimizations. asm statements that have no output
 operands, including asm goto statements, are implicitly volatile.

This i386 code demonstrates a case that does not use (or require) the
volatile qualifier. If it is performing assertion checking, this code
uses asm to perform the validation. Otherwise, dwRes is unreferenced by
any code. As a result, the optimizers can discard the asm statement, which
in turn removes the need for the entire DoCheck routine. By omitting the
volatile qualifier when it isn’t needed you allow the optimizers to produce
the most efficient code possible.

void DoCheck(uint32_t dwSomeValue)
{
   uint32_t dwRes;

   // Assumes dwSomeValue is not zero.
   asm ("bsfl %1,%0"
     : "=r" (dwRes)
     : "r" (dwSomeValue)
     : "cc");

   assert(dwRes > 3);
}
The next example shows a case where the optimizers can recognize that
the input (dwSomeValue) never changes during the execution of the function
and can therefore move the asm outside the loop to produce more efficient
code. Again, using volatile disables this type of optimization.

void do_print(uint32_t dwSomeValue)
{
   uint32_t dwRes;

   for (uint32_t x=0; x < 5; x++)
   {
      // Assumes dwSomeValue is not zero.
      asm ("bsfl %1,%0"
        : "=r" (dwRes)
        : "r" (dwSomeValue)
        : "cc");

      printf("%u: %u %u\n", x, dwSomeValue, dwRes);
   }
}
The following example demonstrates a case where you need to use the
volatile qualifier. It uses the x86 rdtsc instruction, which reads
the computer’s time-stamp counter. Without the volatile qualifier,
the optimizers might assume that the asm block will always return the
same value and therefore optimize away the second call.
如果不加volatile,编译器可能会认为这段asm code block返回的值不会改变，然后
优化的第二个asm block. 可以通过gcc -O3来验证:).

uint64_t msr;

asm volatile ( "rdtsc\n\t"    // Returns the time in EDX:EAX.
        "shl $32, %%rdx\n\t"  // Shift the upper bits left.
        "or %%rdx, %0"        // 'Or' in the lower bits.
        : "=a" (msr)
        :
        : "rdx");

printf("msr: %llx\n", msr);

// Do other work...

// Reprint the timestamp
asm volatile ( "rdtsc\n\t"    // Returns the time in EDX:EAX.
        "shl $32, %%rdx\n\t"  // Shift the upper bits left.
        "or %%rdx, %0"        // 'Or' in the lower bits.
        : "=a" (msr)
        :
        : "rdx");

printf("msr: %llx\n", msr);
GCC’s optimizers do not treat this code like the non-volatile code in the
earlier examples. They do not move it out of loops or omit it on the assumption
that the result from a previous call is still valid.

Note that the compiler can move even volatile asm instructions relative to
other code, including across jump instructions. For example, on many targets
there is a system register that controls the rounding mode of floating-point
operations. Setting it with a volatile asm, as in the following PowerPC
example, does not work reliably.

asm volatile("mtfsf 255, %0" : : "f" (fpenv));
sum = x + y;
The compiler may move the addition back before the volatile asm. To make it
work as expected, add an artificial dependency to the asm by referencing a
variable in the subsequent code, for example:

asm volatile ("mtfsf 255,%1" : "=X" (sum) : "f" (fpenv));
sum = x + y;
Under certain circumstances, GCC may duplicate (or remove duplicates of) your
assembly code when optimizing. This can lead to unexpected duplicate symbol
errors during compilation if your asm code defines symbols or labels.
Using ‘%=’ (see AssemblerTemplate) may help resolve this problem.
```

Here there is another posts: https://gcc.gnu.org/ml/gcc-help/2011-04/msg00174.html
But in latest extend.text, I could not find this piece of doc.
```
Index: extend.texi
===================================================================
--- extend.texi	(revision 172305)
+++ extend.texi	(working copy)
@@ -5784,8 +5784,20 @@

 Similarly, you can't expect a
 sequence of volatile @code{asm} instructions to remain perfectly
-consecutive.  If you want consecutive output, use a single @code{asm}.
-Also, GCC will perform some optimizations across a volatile @code{asm}
+consecutive, meaning that the different volatile @code{asm} instructions
+might get interspersed with assembler instructions from other code parts.
+The volatile @code{asm} instructions will not be reordered though, so the
+volatile @code{asm} instructions will appear in the same order in the
+assembler output as they appear in the source code.
+If you want completely consecutive output (not interspersed with
+assembler instructions from other code parts), use a single @code{asm}.
+volatile @code{asm} instructions are treated in a similar
+manner as accesses to volatile objects. That means that
+volatile @code{asm} instructions will not be moved across accesses to
+volatile objects, so all accesses to volatile objects and all
+volatile @code{asm} instructions will stay in the order as defined
+by the source code.
+GCC will perform some optimizations across a volatile @code{asm}
 instruction; GCC does not ``forget everything'' when it encounters
 a volatile @code{asm} instruction the way some other compilers do.
```
