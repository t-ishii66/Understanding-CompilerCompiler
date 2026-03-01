# Understanding `comcom.h`

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

From here on, we examine the runtime mechanisms in `comcom.h`. There are four main themes.

- rewinding a branch with `setjmp` / `longjmp`
- controlling whitespace skipping with `combine()`
- backtracking through `back()`
- the roles of `print()`, `genNum()`, `spoolOpen()`, and `spoolClose()`

## The World of `longjmp`

There is a C feature that is not used very often: `setjmp()` and `longjmp()`. The behavior itself is not difficult. Start with an example.

<div class="codeblock-title">Sample code using `longjmp`</div>

```c
#include <setjmp.h>
int main()
{
    jmp_buf a;
    if (!setjmp(a)) {
        printf("before long jump\n");
        longjmp(a, 1);
        printf("after long jump\n");
    } else {
        printf("else statement\n");
    }
}
```

<div class="codeblock-title">Output</div>

```text
before long jump
else statement
```

When `longjmp()` is called, execution returns to the matching `setjmp()`.

- The first time `setjmp(a)` is called, it returns 0
- When control returns there via `longjmp(a, 1)`, `setjmp(a)` returns 1

This behavior looks a little strange, but the essential idea is simple: `setjmp()` saves the current execution state, and `longjmp()` jumps back to it.

However, not everything is restored automatically. Global variables and similar state must be restored separately. That is the important point in `comcom.h`.

First, look at the definitions of `try()` and `ok()`.

<div class="codeblock-title">Definition of `try()` / `ok()`</div>

```c
#define try() save(),!setjmp(BS[B++].j)
#define ok() B--
```

What appears here for the first time is the backtracking stack `BS[]`.

## The Backtracking Stack

The backtracking stack `BS[]` is an array used to save the information needed to return via `longjmp()`.

<div class="codeblock-title">Definition of the backtracking stack</div>

```c
struct {
    jmp_buf j;
    int e;
    int t;
    int i;
    int k;
} BS[BS_SIZE];
```

- `j` stores the execution environment for `setjmp()`
- `e`, `t`, `i`, and `k` store state that `jmp_buf` alone cannot restore

Here `e` and `t` save the global variables `E` and `T`, which we already saw as the current positions in `ES[]` and `TS[]`.

That leaves `i` and `k`.

## Global Variables `I` and `K`

The value saved in `i` is the global variable `I`. `I` is the current position in the input file.

The entire input file is first read into `IB[]` inside `init()`, and after that `fileInput()` returns it one character at a time.

<div class="codeblock-title">comcom.h (simplified `fileInput`)</div>

```c
uint fileInput()
{
    return (uint)((I < max_I) ? IB[I ++] : EOF);
}
```

`I` is the position to be read next.

The other variable, `K`, is used by the following macro.

<div class="codeblock-title">comcom.h (macro `combine`)</div>

```c
#define combine() K=FALSE
```

This `K` is a flag used to temporarily disable whitespace skipping.

## Forbidding Whitespace Skipping

In this system, whitespace characters in the input are skipped by default. In other words, the following two inputs are treated as equivalent:

```text
This is a pen
```

```text
This   is a   pen
```

By default, `K = TRUE`. In that state, look at `skipSpace()`.

<div class="codeblock-title">comcom.h (`skipSpace` function)</div>

```c
skipSpace(byte s)
{
    uint c;
    if (!K) {K = TRUE; return;}
    while ((c = fileInput()) != EOF && (byte)c != s &&
isspace(c))
        ;
    -- I;
}
```

When `K = TRUE`, it skips whitespace until just before the comparison target character `s`.

The function that calls `skipSpace()` is `check()`.

<div class="codeblock-title">comcom.h (`check` function)</div>

```c
check(byte* s)
{
    skipSpace(*s);
    while (*s) {
        char c = fileInput();
        if (*s ++ != c)
            back();
    }
}
```

For example, if we call `check("This")`, then `skipSpace('T')` runs first. That means whitespace is skipped until the initial `T`, and after that the characters in `This` are compared one by one.

The interesting case is something like `check(" is a ")`, where the comparison target itself begins with a space. In that case `skipSpace(' ')` is called, but because the target character is itself a space, that space is not skipped.

This is why strings containing spaces can still be handled correctly.

Now for the role of `K`. When `combine()` is called, `K` becomes `FALSE`. When the next `check()` runs in that state, `skipSpace()` does not skip spaces and simply returns immediately. But this effect lasts for only one call: after returning, `K` is immediately restored to `TRUE`.

This matters when a string is compared one character at a time.

For example, in the self-definition discussed later, there are situations where we cannot write `check("This")`, and must instead use:

```c
check("T"); check("h"); check("i"); check("s");
```

If we leave it like that, whitespace is skipped for each `check()`, so both of the following inputs would match:

```text
This
```

```text
T h  i s
```

To prevent this, insert `combine()` between the character checks.

```c
check("T"); combine(); check("h"); combine(); check("i"); combine(); check("s");
```

Now whitespace is forbidden between `T`, `h`, `i`, and `s`. So semantically this becomes equivalent to `check("This")`.

With that mechanism in mind, look at a one-character-at-a-time comparison example.

<div class="codeblock-title">PEN1.def</div>

```c
START(PEN1)
PEN1 : A B C { call(1); call(2); call(3); }
;
A : "A" { print("a"); }
;
B : "B" { print("b"); }
;
C : "C" { print("c"); }
;
END
```

This outputs `abc` whether the input is `ABC` or `A B C`.

Now add `+`.

<div class="codeblock-title">PEN2.def</div>

```c
START(PEN2)
PEN2 : A+B+C { call(1); call(2); call(3); }
;
A : "A" { print("a"); }
;
B : "B" { print("b"); }
;
C : "C" { print("c"); }
;
END
```

This version outputs `abc` only when the input is `ABC`. `A B C` becomes an error.

If you compare the C source generated by `ccgen` for these two definitions, the only difference is whether `combine();` appears.

<table class="plain-table">
<colgroup>
<col style="width: 50%" />
<col style="width: 50%" />
</colgroup>
<tbody>
<tr>
<td>PEN1.c</td>
<td>PEN2.c</td>
</tr>
<tr>
<td>

```c
#include "comcom.h"

void firstcall(int e) { _PEN1(e); }

void _PEN1(int e) {

{ _A(E); _B(E); _C(E); mknode(_f1);}

}

void _A(int e) {

{ check("A"); mknode(_f2);}

}

void _B(int e) {

{ check("B"); mknode(_f3);}

}

void _C(int e) {

{ check("C"); mknode(_f4);}

}

void _f1(int n, int d) {call(1);call(2);call(3);}

void _f2(int n, int d) {print("a");}

void _f3(int n, int d) {print("b");}

void _f4(int n, int d) {print("c");}
```

</td>
<td>

```c
#include "comcom.h"

void firstcall(int e) { _PEN2(e); }

void _PEN2(int e) {

{ _A(E); combine(); _B(E); combine(); _C(E); mknode(_f1);}

}

void _A(int e) {

{ check("A"); mknode(_f2);}

}

void _B(int e) {

{ check("B"); mknode(_f3);}

}

void _C(int e) {

{ check("C"); mknode(_f4);}

}

void _f1(int n, int d) {call(1);call(2);call(3);}

void _f2(int n, int d) {print("a");}

void _f3(int n, int d) {print("b");}

void _f4(int n, int d) {print("c");}
```

</td>
</tr>
</tbody>
</table>

So when `+` is used in a syntax rule, the generated C code inserts `combine();`. That is what "compare continuously without skipping whitespace" means.

## Backtracking

Next is backtracking. When `check()` finds a mismatch, it calls `back()`. This is the mechanism that rewinds processing to the most recent branch point and tries another candidate.

Start with an example.

<div class="codeblock-title">Sample code for understanding backtracking</div>

```c
#include "comcom.h"
int main()
{
    E=T=0;
    A(E);
    exec();
}
A(int e)
{
    if (try()) {
        check("a"); mknode(_f1); ok();
    } else if (try()) {
        check("b"); mknode(_f2); ok();
    } else {
        check("c"); mknode(_f3);
    }
}
_f1(int n, int d) { printf("This is a\n"); }
_f2(int n, int d) { printf("This is b\n"); }
_f3(int n, int d) { printf("This is c\n"); }
```

Here again are `try()` and `ok()`, along with a simplified `save()`.

<div class="codeblock-title">Definitions of `try()` / `ok()` and a simplified `save()`</div>

```c
#define try() save(),!setjmp((BS[B++].j))
#define ok() B--
void save()
{
    if (B >= BS_SIZE)
        abend("BS over");
    BS[B].e = E;
    BS[B].t = T;
    BS[B].i = I;
    BS[B].k = K;
    if (B >= maxB)
        maxB = B + 1;
}
```

`save()` is simple: it saves the current `E`, `T`, `I`, and `K` into `BS[]`.

`try()` performs that save and then executes `setjmp()`. The first `setjmp()` returns 0, so `!setjmp(...)` is true and execution enters the body of `if (try()) { ... }`.

If the input is `a`, then the first `check("a")` succeeds, `mknode(_f1)` runs, and finally `ok()` performs `B--`. That releases the backtracking stack entry reserved for that `try()`.

Now consider the case where the input is `b`.

The first `check("a")` fails, so `back()` is called.

<div class="codeblock-title">comcom.h (`back` function)</div>

```c
back()
{
    if (--B < 0)
        abend("syntax");
    E = BS[B].e;
    T = BS[B].t;
    I = BS[B].i;
    K = BS[B].k;
    ++ cnt_B;
    longjmp(&(BS[B].j), 1);
}
```

`back()` restores the `E`, `T`, `I`, and `K` that were saved by the most recent `try()`, and then uses `longjmp()` to return to the corresponding `setjmp()`.

After the jump back, `setjmp()` returns 1, so `!setjmp(...)` becomes false. As a result, the first `if (try())` fails, and control moves on to the next `else if (try())`.

There `check("b")` runs. This time it succeeds, so `_f2` is pushed into the parse tree `TS[]`.

That is what backtracking does: it rewinds the state from the point of failure back to the most recent branch point, and then tries another candidate. `try()`, `back()`, and `ok()` work as a set.

## Other Functions and Macros: `print`, `genNum`, `spoolOpen`, `spoolClose`

There are not many remaining functions and macros.

`print(...)` writes a string to the current output file. Here "current output file" means either the real output file specified when running `ccgen`, or a temporary file.

For convenience, let us call them:

- the real output file: `OUTFILE` (the second argument to `ccgen`)
- the temporary file: `TMPFILE`

By default, the output destination is `OUTFILE`. When `spoolOpen()` is called, the destination switches to `TMPFILE`. When `spoolClose()` is called, it switches back to `OUTFILE`.

After all processing is complete, the contents of `TMPFILE` are appended to the end of `OUTFILE`, and then `TMPFILE` itself is deleted.

<div class="codeblock-title">Sample code</div>

```c
print("This is ");
spoolOpen();
print("---");
print("END!\n");
spoolClose();
print("a pen.\n");
```

<div class="codeblock-title">Output (`OUTFILE` contents)</div>

```text
This is a pen.
---END!
```

Next is `genNum()`.

<div class="codeblock-title">comcom.h (`genNum()` macro and `genNumber()` function)</div>

```c
#define genNum() genNumber(&d)
void genNumber(uint* d)
{
    static char s[6];
    if (*d == 0)
        *d = ++ D;
    sprintf(s, "%u", *d);
    print(s);
}
```

`genNum()` expands to `genNumber(&d)`. If `d` is 0, a new number is taken from `D`, and after that the same `d` is reused.

Consider this example.

<div class="codeblock-title">Sample code</div>

```c
main(){ D=3; sample(0); }
sample(int d) { genNumber(&d); print("=");genNumber(&d); }
```

The output is:

```text
4=4
```

That is because the first `genNumber(&d)` sets `d` to 4, and the second call reuses that same 4.

The important point here is that `genNum()` written in a definition file and `genNum()` running in the generated C code belong to different stages.

The `genNum()` written in a generation rule in the definition file is not a function that runs on the spot. It is only a pattern recognized by `ccgen`. When `ccgen` finds it, it writes the literal text `genNum();` into the generated C source.

Here is an example.

<div class="codeblock-title">SAMPLE1.def</div>

```c
START(SAMPLE1)
SAMPLE1 : "test" { genNum(); print("="); genNum(); }
;
END
```

The key point is the same as with `+` earlier. Neither `+` nor `genNum()` is an instruction that executes at the moment it appears in the definition file. Both are descriptions that tell `ccgen` what to write into the generated C source. The only difference is where they appear. `+` appears on the syntax-rule side, so `ccgen` outputs `combine()`. `genNum()` appears on the generation-rule side, so `ccgen` outputs `genNum();`.

The C source emitted by `ccgen` looks like this:

<div class="codeblock-title">`SAMPLE1.c` generated by `ccgen`</div>

```c
#include "comcom.h"
void firstcall(int e) { _SAMPLE1(e); }
void _SAMPLE1(int e) {
    { check("test"); mknode(_f1);}
}
void _f1(int n, int d) {genNum();print("=");genNum();}
```

`_f1(n, d)` is always called through `call2(n, b)`, and at that moment `d` is always 0. So in this example the first `genNum()` takes a fresh number, and the second one uses the same number again.

Since the initial state has `D = 0`, if the input is `test`, the output becomes `1=1`.

In short, any `genNum();` calls that occur within one generation rule always produce the same number, but `genNum();` in a different generation rule will never share that same number.

`print()` follows the same idea. A `print(...)` written in a generation rule of the definition file is not a function call with immediate meaning at that stage. It is text that `ccgen` copies directly into the C source it generates. The actual execution happens later, when the generated C program runs using `comcom.h`.
