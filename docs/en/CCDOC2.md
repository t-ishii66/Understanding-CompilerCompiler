---
title: "Structure of the Generated File"
description: "Understand the structure of output files produced by a compiler-compiler: how grammar rules are translated into executable C code."
lang: en
---

# Structure of the Generated File

<style>
body, p, li, h1, h2, h3, h4, h5, h6, td, th {
  font-family: "SFMono-Regular", Menlo, Consolas, "Liberation Mono", "Courier New", "Hiragino Sans", "Yu Gothic", "Noto Sans JP", monospace;
  font-variant-ligatures: none;
}

code, pre, pre code, .codeblock-title {
  font-family: "SFMono-Regular", Menlo, Consolas, "Liberation Mono", "Courier New", monospace;
  font-variant-ligatures: none;
}

table {
  width: 100%;
  border-collapse: collapse;
  border: 1px solid #f0dc9f;
  background: #ffffff;
  font-family: inherit;
  font-size: 0.95em;
}
table thead th,
table thead td {
  background: #fff3bf;
  color: #5c4a00;
  font-weight: 700;
}
table td,
table th {
  border: 1px solid #f0dc9f;
  padding: 6px 8px;
  vertical-align: top;
}
table tbody tr td,
table tbody tr th {
  background: #eef6ff !important;
}
table tbody tr:nth-child(even) td,
table tbody tr:nth-child(even) th {
  background: #e3efff !important;
}

table.plain-table,
table.plain-table thead th,
table.plain-table thead td,
table.plain-table tbody td,
table.plain-table tbody th,
table.plain-table td,
table.plain-table th,
table.plain-table tbody tr td,
table.plain-table tbody tr th,
table.plain-table tbody tr:nth-child(even) td,
table.plain-table tbody tr:nth-child(even) th,
table.plain-table tbody tr:nth-child(odd) td,
table.plain-table tbody tr:nth-child(odd) th {
  background: #ffffff !important;
}

table.plain-table pre {
  background: #ffffff;
  border: 0;
}

table.plain-table pre code {
  background: #ffffff;
  color: #000000;
}

table.plain-table .codeblock-title {
  background: #ffffff;
  border: 0;
  margin: 0;
  padding: 0 0 8px 0;
}
pre {
  background: #ffffff;
  border: 1px solid #000000;
  padding: 10px 12px;
  overflow-x: auto;
}
pre code {
  background: transparent;
  color: #000000;
}
.codeblock-title {
  display: block;
  font-family: inherit;
  font-size: 0.95em;
  background: #ffffff;
  border: 1px solid #000000;
  border-bottom: 0;
  padding: 8px 12px;
  margin: 0 0 -1px 0;
}
.codeblock-title + pre {
  margin-top: 0;
}
</style>

First, the C source emitted by `ccgen` always begins like this.

<div class="codeblock-title">PEN.c (excerpt)</div>

```c
#include "comcom.h"
void firstcall(int e) { _PEN0(e); }
```

The generated source runs on top of `comcom.h`. `main()` is also defined in `comcom.h`, and in the end it calls `firstcall()`.

To understand what follows, you need to keep two arrays in mind: the parse tree `TS[]` and the entry stack `ES[]`.

# The Parse Tree and the Entry Stack

The core of this system is the parse tree `TS[]` and the entry stack `ES[]`.

- `TS[]` is an array that stores either integers or function pointers
- `ES[]` is an array that stores only integers

`TS[]` represents the tree itself. `ES[]` is used to point to nodes that have not yet been grouped under a parent.

## Pointers to Functions

If function pointers are unfamiliar, first look only at this example.

<div class="codeblock-title">Example of a function pointer</div>

```c
#include <stdio.h>
int func(int a, int b)
{
    return a + b;
}
int main()
{
    int ret;
    int (*f)(int, int);
    f = func;
    ret = (*f)(3, 4);  /* calls func; ret becomes 7 */
}
```

A function pointer is a kind of variable. Its declaration has this shape:

```c
return_type (*name)(arguments);
```

For example, a function `func` that takes two integers and returns an integer is declared like this:

```c
int func(int, int);
```

The corresponding function pointer is:

```c
int (*f)(int, int);
```

When you write `f = func;`, the address of `func` is assigned to `f`. The function is not called yet at that point. The call itself is this:

```c
ret = (*f)(2, 3);
```

`TS[]` stores function pointers of this kind.

## The Parse Tree `TS[]`

Because `PEN.c` includes `comcom.h`, the real structure lives there. Start with `TS[]`.

<div class="codeblock-title">comcom.h (the structure used to build the parse tree)</div>

```c
union {
    int (*f)();
    int n;
} TS[TS_SIZE];
```

The main point is simple: each element of `TS[]` contains either a function pointer or an integer.

| index | contents of parse tree TS[] | access form |
|------|-------------------------------|-------------|
| 0 | function pointer | TS[0].f |
| 1 | integer | TS[1].n |
| 2 | integer | TS[2].n |
| 3 | function pointer | TS[3].f |
| 4 | integer | TS[4].n |
| ... | ... | ... |

## The Entry Stack `ES[]`

<div class="codeblock-title">comcom.h (the structure used to build the entry stack)</div>

```c
int ES[ES_SIZE];
```

This one is just a plain integer array.

| index | contents of entry stack ES[] | access form |
|------|---------------------------------|-------------|
| 0 | integer | ES[0] |
| 1 | integer | ES[1] |
| 2 | integer | ES[2] |
| 3 | integer | ES[3] |
| ... | ... | ... |

The parse tree `TS[]` is built through `makeNode()` and `mknode()`.

<div class="codeblock-title">comcom.h (operations related to the parse tree and entry stack)</div>

```c
void storeFunc(int (*f)())
{
    if (T > TS_SIZE)
        abend("TS over");
    TS[T ++].f = f;
    if (T > maxT)
        maxT = T;
}
void storeInt(int n)
{
    if (T >= TS_SIZE)
        abend("TS over");
    TS[T ++].n = n;
    if (T > maxT)
        maxT = T;
}
#define mknode(f) { void f(int,int); makeNode(f, e); }
void makeNode(void (*n)(int,int), int e)
{
    int i, t = T;
    storeFunc(n);
    storeInt(E - e);
    for (i = e ; i < E ; ++ i) {
        storeInt(ES[i]);
    }
    E = e;
    if (E >= ES_SIZE)
        abend("ES over");
    ES[E ++] = t;
    if (E > maxE)
        maxE = E;
}
```

This is the first difficult point. Do not try to understand it all at once. Follow it through a concrete example.

## Global Variables `T` and `E`

`TS[]` and `ES[]` are used from the front onward. `T` and `E` are the current indices into `TS[]` and `ES[]`.

- `T` is the next free position in `TS[]`
- `E` is the next free position in `ES[]`

Both start at 0. Every time one element of `TS[]` is used, `T` is incremented by 1. Every time one element of `ES[]` is used, `E` is incremented by 1.

In the following sample, let us see how a parse tree `TS[]` is actually built.

<div class="codeblock-title">Sample code for understanding parse tree `TS[]` and entry stack `ES[]`</div>

```c
main() {
    E = T = 0;
    A(E);
    exec();
}
A(int e) { B(E); C(E); mknode(_f1); }
B(int e) { D(E); F(E); mknode(_f2); }
C(int e) { mknode(_f3); }
D(int e) { mknode(_f4); }
F(int e) { mknode(_f5); }
_f1(int n, int d) { call(1); call(2); }
_f2(int n, int d) { call(1); call(2); }
_f3(int n, int d) { printf("World\n"); }
_f4(int n, int d) { printf("Hello"); }
_f5(int n, int d) { printf(", "); }
```

It looks simple, but `mknode(...)` and `call(...)` are the important parts.

When `main()` calls `A(E)`, we still have `E = 0`. So inside `A`, the argument `e` is 0. `A` first calls `B(E)`. Since `E` is still 0, this means `B(0)` is called.

`B` then calls `D(E)`. So the first call to trace is `D(0)`.

The body of `D` is only `mknode(_f4);`. This is a macro:

```c
#define mknode(f) { void f(int,int); makeNode(f, e); }
```

Inside `D(0)`, `e = 0`, so what actually runs is:

```c
makeNode(_f4, 0);
```

Here is `makeNode()` again.

<div class="codeblock-title">comcom.h (function `makeNode`)</div>

```c
void makeNode(void (*n)(int,int), int e)
{
    int i, t = T;
    storeFunc(n);
    storeInt(E - e);
    for (i = e ; i < E ; ++ i) {
        storeInt(ES[i]);
    }
    E = e;
    if (E >= ES_SIZE)
        abend("ES over");
    ES[E ++] = t;
    if (E > maxE)
        maxE = E;
}
```

At the moment of the call, the relevant variables are:

| variable handled by makeNode | at start of makeNode | after makeNode finishes (for reference) |
|--------------------------------|------------------------|-------------------------------------------|
| n | _f4 | _f4 |
| e | 0 | 0 |
| t | 0 | 0 |
| T | 0 | 2 |
| E | 0 | 1 |

First, `storeFunc(_f4)` runs, so `_f4` is pushed into `TS[]`, and `T` becomes 1.

<div class="codeblock-title">comcom.h (function `storeFunc`)</div>

```c
void storeFunc(int (*f)())
{
    TS[T ++].f = f;
}
```

| index | parse tree TS[] | value of T | value of E | entry stack ES[] |
|------|--------------------|--------------|--------------|--------------------|
| 0 | _f4 |  | ○ |  |
| 1 |  | ○ |  |  |

Next, `storeInt(E - e)` runs. Since `E = 0` and `e = 0`, this is `storeInt(0)`.

<div class="codeblock-title">comcom.h (function `storeInt`)</div>

```c
storeInt(int n) {
    TS[T++].n = n;
}
```

| index | parse tree TS[] | value of T | value of E | entry stack ES[] |
|------|--------------------|--------------|--------------|--------------------|
| 0 | _f4 |  | ○ |  |
| 1 | 0 |  |  |  |
| 2 |  | ○ |  |  |

The meaning of this `0` will become clear in a moment.

At this point, `E = e = 0`, so the loop `for (i = e; i < E; ++i)` does not execute even once. The next line `E = e;` also changes nothing.

Finally, `ES[E ++] = t;` runs. Here `t = 0`, so `ES[0] = 0` is stored.

| index | parse tree TS[] | value of T | value of E | entry stack ES[] |
|------|--------------------|--------------|--------------|--------------------|
| 0 | _f4 |  |  | 0 |
| 1 | 0 |  | ○ |  |
| 2 |  | ○ |  |  |

This finishes `mknode(_f4);`. The result is `T = 2`, `E = 1`.

Next comes `F(E)` inside `B(0)`. At this point `E = 1`, so what actually gets called is `F(1)`. In other words, inside `F`, the argument `e` is 1.

The body of `F` is only `mknode(_f5);`, so `makeNode(_f5, 1);` is executed. At the moment of that call:

| variable handled by makeNode | at start of makeNode | after makeNode finishes (for reference) |
|--------------------------------|------------------------|-------------------------------------------|
| n | _f5 | _f5 |
| e | 1 | 1 |
| t | 2 | 2 |
| T | 2 | 4 |
| E | 1 | 2 |

First, `_f5` is pushed into `TS[]`.

| index | parse tree TS[] | value of T | value of E | entry stack ES[] |
|------|--------------------|--------------|--------------|--------------------|
| 0 | _f4 |  |  | 0 |
| 1 | 0 |  | ○ |  |
| 2 | _f5 |  |  |  |
| 3 |  | ○ |  |  |

Then `storeInt(E - e)` runs. Now `E = 1` and `e = 1`, so again 0 is pushed.

| index | parse tree TS[] | value of T | value of E | entry stack ES[] |
|------|--------------------|--------------|--------------|--------------------|
| 0 | _f4 |  |  | 0 |
| 1 | 0 |  | ○ |  |
| 2 | _f5 |  |  |  |
| 3 | 0 |  |  |  |
| 4 |  | ○ |  |  |

Again, the `for` loop does not run. Finally, `ES[E ++] = t;` stores `t = 2` into `ES[]`.

| index | parse tree TS[] | value of T | value of E | entry stack ES[] |
|------|--------------------|--------------|--------------|--------------------|
| 0 | _f4 |  |  | 0 |
| 1 | 0 |  |  | 2 |
| 2 | _f5 |  | ○ |  |
| 3 | 0 |  |  |  |
| 4 |  | ○ |  |  |

At this point, the `0` and `2` stored in `ES[]` point to the positions of `_f4` and `_f5` inside `TS[]`.

Now return to `B` and execute the final `mknode(_f2);`. Since `B` was called as `B(0)` from `A`, the value of `e` inside `B` is still 0. So what actually runs is `makeNode(_f2, 0);`.

At the moment of that call:

| variable handled by makeNode | at start of makeNode | after makeNode finishes (for reference) |
|--------------------------------|------------------------|-------------------------------------------|
| n | _f2 | _f2 |
| e | 0 | 0 |
| t | 4 | 4 |
| T | 4 | 8 |
| E | 2 | 1 |

First, `_f2` is pushed.

| index | parse tree TS[] | value of T | value of E | entry stack ES[] |
|------|--------------------|--------------|--------------|--------------------|
| 0 | _f4 |  |  | 0 |
| 1 | 0 |  |  | 2 |
| 2 | _f5 |  | ○ |  |
| 3 | 0 |  |  |  |
| 4 | _f2 |  |  |  |
| 5 |  | ○ |  |  |

Next, `storeInt(E - e)` runs. Here `E = 2` and `e = 0`, so 2 is pushed.

This `E - e` means "how many `ES[]` entries were consumed during execution of this function". Inside `B`, two entries were pushed by `D(E)` and `F(E)`, so the value here is 2.

After that, `for (i = e; i < E; ++i)` copies the values currently stored in `ES[]` into `TS[]`. In this case, `0` and `2` are copied.

| index | parse tree TS[] | value of T | value of E | entry stack ES[] |
|------|--------------------|--------------|--------------|--------------------|
| 0 | _f4 |  |  | 0 (points to _f4) |
| 1 | 0 |  |  | 2 (points to _f5) |
| 2 | _f5 |  | ○ |  |
| 3 | 0 |  |  |  |
| 4 | _f2 |  |  |  |
| 5 | 2 |  |  |  |
| 6 | 0 (points to _f4) |  |  |  |
| 7 | 2 (points to _f5) |  |  |  |
| 8 |  | ○ |  |  |

What this shows is that one node in `TS[]` has the following shape:

```c
[function, number_of_branches, branch1, branch2, ...]
```

In the case of `_f2`, it becomes a node whose branches are `_f4` and `_f5`.

The following `E = e;` rewinds the portion of `ES[]` that was consumed while executing `B`. Here `E` becomes 0, so the two entries used by `B` are released.

Finally, `ES[E ++] = t;` runs. Here `t = 4`, which is the position where `_f2` was first placed in `TS[]`. In other words, `ES[]` is left with exactly one value: a pointer to `_f2`.

| index | parse tree TS[] | value of T | value of E | entry stack ES[] |
|------|--------------------|--------------|--------------|--------------------|
| 0 | _f4 |  |  | 4 (points to _f2) |
| 1 | 0 |  | ○ |  |
| 2 | _f5 |  |  |  |
| 3 | 0 |  |  |  |
| 4 | _f2 |  |  |  |
| 5 | 2 |  |  |  |
| 6 | 0 (points to _f4) |  |  |  |
| 7 | 2 (points to _f5) |  |  |  |
| 8 |  | ○ |  |  |

At this point the structure is becoming visible.

- Nodes are accumulated in `TS[]`
- `ES[]` keeps only the nodes that have not yet been grouped under a parent
- When one function finishes, its effect on `ES[]` is always reduced to exactly one entry

Now return to `A` and execute `C(E)`. At this point `E = 1`, so `C(1)` is called. The body of `C` is only `mknode(_f3);`.

At the moment of that call:

| variable handled by makeNode | at start of makeNode | after makeNode finishes (for reference) |
|--------------------------------|------------------------|-------------------------------------------|
| n | _f3 | _f3 |
| e | 1 | 1 |
| t | 8 | 8 |
| T | 8 | 10 |
| E | 1 | 2 |

`C` does not consume anything from `ES[]`, so `storeInt(E - e)` pushes 0, and in the end one more value pointing to `_f3` is added to `ES[]`.

The result is:

| index | parse tree TS[] | value of T | value of E | entry stack ES[] |
|------|--------------------|--------------|--------------|--------------------|
| 0 | _f4 |  |  | 4 (points to _f2) |
| 1 | 0 |  |  | 8 (points to _f3) |
| 2 | _f5 |  | ○ |  |
| 3 | 0 |  |  |  |
| 4 | _f2 |  |  |  |
| 5 | 2 |  |  |  |
| 6 | 0 (points to _f4) |  |  |  |
| 7 | 2 (points to _f5) |  |  |  |
| 8 | _f3 |  |  |  |
| 9 | 0 |  |  |  |
| 10 |  | ○ |  |  |

Finally, execute `mknode(_f1);` in `A`. Since `A` was called from `main()` as `A(0)`, we have `e = 0`. So `makeNode(_f1, 0);` is executed.

At the moment of that call:

| variable handled by makeNode | at start of makeNode | after makeNode finishes (for reference) |
|--------------------------------|------------------------|-------------------------------------------|
| n | _f1 | _f1 |
| e | 0 | 0 |
| t | 10 | 10 |
| T | 10 | 14 |
| E | 2 | 1 |

Again, `E - e = 2`, so the two nodes created inside `A`, namely `_f2` and `_f3`, are grouped as branches under `_f1`.

The final state is:

| index | parse tree TS[] | value of T | value of E | entry stack ES[] |
|------|--------------------|--------------|--------------|--------------------|
| 0 | _f4 |  |  | 10 (points to _f1) |
| 1 | 0 |  | ○ |  |
| 2 | _f5 |  |  |  |
| 3 | 0 |  |  |  |
| 4 | _f2 |  |  |  |
| 5 | 2 |  |  |  |
| 6 | 0 (points to _f4) |  |  |  |
| 7 | 2 (points to _f5) |  |  |  |
| 8 | _f3 |  |  |  |
| 9 | 0 |  |  |  |
| 10 | _f1 |  |  |  |
| 11 | 2 |  |  |  |
| 12 | 4 (points to _f2) |  |  |  |
| 13 | 8 (points to _f3) |  |  |  |
| 14 |  | ○ |  |  |

Now the parse tree `TS[]` is complete. The job of `firstcall(E)` as invoked from `main()` was to build this tree.

When construction finishes, exactly one value remains in `ES[]`. It points to the root of the parse tree `TS[]`. If the result is not in that shape, it is a syntax error.

## Executing the Parse Tree `TS[]`

Next, execute the parse tree `TS[]` that was just built. After `firstcall(E)`, `main()` calls `exec()`.

<div class="codeblock-title">comcom.h (simplified version of `exec`)</div>

```c
exec() {
    int t;
    t = ES[-- E];
    (*TS[t].f)(t, 0);
}
```

The real code also checks that `E == 1`. For the moment, consider this simplified version.

The value left in `ES[]` is 10, so `t = 10`. Therefore this runs:

```c
(*TS[10].f)(10, 0);
```

`TS[10].f` is `_f1`, so in the end this means calling `_f1(10, 0)`.

<div class="codeblock-title">Source code of function `_f1`</div>

```c
_f1(int n, int d) {
    call(1); call(2);
}
```

`call` is a macro whose body is:

<div class="codeblock-title">comcom.h (macro `call`)</div>

```c
#define call(a) call2(n, a)
```

So `_f1(10, 0)` is effectively:

```c
call2(10, 1); call2(10, 2);
```

This is where `call2()` comes in.

<div class="codeblock-title">comcom.h (function `call2`)</div>

```c
void call2(int n, int b)
{
    int m;
    if (b > TS[n + 1].n || b < 1)
        abend("branch no");
    m = (TS[n + 1 + b].n);
    (*TS[m].f)(m, 0);
}
```

The important part here is the calculation `TS[n + 1 + b].n`. It is easier to see if you imagine the following macro:

```c
#define nb(n, b) (TS[n + 1 + b].n)
```

`n` is the start of the current node, that is, the position of its function pointer. `b` is which branch of that node you want.

For example, in `call2(10, 1)`, `nb(10, 1)` reads `TS[12].n`. That contains 4, so `m = 4`. Therefore the final call becomes:

```c
(*TS[4].f)(4, 0);
```

`TS[4].f` is `_f2`, so in the end `_f2(4, 0)` is called.

<div class="codeblock-title">Source code of function `_f2`</div>

```c
_f2(int n, int d) {
    call(1);
    call(2);
}
```

The same thing happens again.

- `call2(4, 1)` obtains 0 through `nb(4, 1)`, so it calls `_f4(0, 0)`
- `call2(4, 2)` obtains 2 through `nb(4, 2)`, so it calls `_f5(2, 0)`

`_f4(0, 0)` outputs `"Hello"`, and `_f5(2, 0)` outputs `", "`.

Then control returns to the original `_f1(10, 0)`, and the remaining `call2(10, 2)` runs. This obtains 8 through `nb(10, 2)`, so it calls `_f3(8, 0)`. `_f3(8, 0)` outputs `"World\n"`.

So the final output is:

```text
Hello, World
```

## A Macroscopic Understanding

By now we have traced the detailed behavior of the parse tree `TS[]`, the entry stack `ES[]`, and the behavior of `call2(n, b)`.

But it would be exhausting to trace things this finely every time. Once you understand the mechanism once, a larger view is enough from then on.

<div class="codeblock-title">Macroscopic view</div>

```c
main() {
    E = T = 0;
    A(E);
    exec();
}
A(int e) { B(E); C(E); mknode(_f1); }
B(int e) { D(E); F(E); mknode(_f2); }
C(int e) { mknode(_f3); }
D(int e) { mknode(_f4); }
F(int e) { mknode(_f5); }
_f1(int n, int d) { call(1); call(2); }
_f2(int n, int d) { call(1); call(2); }
_f3(int n, int d) { printf("World\n"); }
_f4(int n, int d) { printf("Hello"); }
_f5(int n, int d) { printf(", "); }
```

Viewed coarsely, `B(E)` first creates `_f4` (via `D`) and `_f5` (via `F`), and then `_f2` (at the end of `B`) groups them together.

```text
_f2 --+--> _f4
      |
      +--> _f5
```

Then control returns to `A(E)`, where `_f1` (at the end of `A`) groups together `_f2` (from `B`) and `_f3` (from `C`).

```text
_f1 --+--> _f2 --+--> _f4
      |          |
      |          +--> _f5
      +--> _f3
```

That is the tree-building phase. At this point, only the value pointing to the root `_f1` remains in `ES[]`.

Next comes the execution phase. Execution starts from the root `_f1`.

- `_f1` calls `_f2` through `call(1)`
- `_f2` calls `_f4` and `_f5` through `call(1)` and `call(2)`
- As a result, `Hello, ` is output
- `_f1` then calls `_f3` through `call(2)`
- As a result, `World\n` is output

So the final output is `Hello, World\n`.

Once you understand the detailed movement once, this macroscopic view is enough from then on. The important picture is simply that `mknode(...)` assembles the tree, and `call(...)` follows its branches and executes it.
