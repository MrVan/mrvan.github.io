
From https://msdn.microsoft.com/en-us/library/f90831hc.aspx

每一个C++表达式要么是rvalue，要么是lvalue. lvalue指向的对象在单个表达式之
外会依旧存在，你可以理解为lvalue是有名字的对象。所有的变量，包括const变量，
都是lvalue。rvalue是临时的，在超出表达式的范围之后就不存在了。

看下面的例子:
```c
#include <iostream>
using namespace std;
int main()
{
   int x = 3 + 4;
   cout << x << endl;
}
```
x是lvalue，因为在表达式`int x = 3 +4`之外，x依旧是存在的(persists beyond the
expression). `3 + 4`是lvalue,因为它does not persists beyond the expression.

继续看下面的例子:
```c
int main()
{
   int i, j, *p;

   // Correct usage: the variable i is an lvalue.
   // i是lvalue
   i = 7;

   // Incorrect usage: The left operand must be an lvalue (C2106).
   //左侧操作数必须是lvalue，7和j*4都不是lvalue.
   7 = i; // C2106
   j * 4 = 7; // C2106

   // Correct usage: the dereferenced pointer is an lvalue.
   //指针解除引用是lvalue
   *p = i;

   const int ci = 7;
   // Incorrect usage: the variable is a non-modifiable lvalue (C3892).
   //const变量是lvalue，但不可以修改
   ci = 9; // C3892

   // Correct usage: the conditional operator returns an lvalue.
   //条件运算符返回一个lvalue.
   ((i < 3) ? i : j) = 7;
}
```
