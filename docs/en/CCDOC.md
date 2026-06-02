---
title: "A Compiler-Compiler from Scratch"
description: "Build a compiler-compiler from the ground up: learn how grammar definitions drive code generation through a hands-on, step-by-step walkthrough."
lang: en
---

# A Compiler-Compiler from Scratch

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

"Compiler-compiler" is not a typo. It means a system that generates compilers. Typical examples are `yacc` and `bison`.

What this document studies is a small hand-built compiler-compiler, and then the mechanism by which that system is defined using its own grammar.

At first this may look roundabout. But if you follow the concrete examples in order, the overall picture becomes visible. The priority here is not to pause on every detail, but to grasp the flow first.

# Goal

The compiler-compiler discussed here is written in C. Its source file is `t3.out.c`, and we will call the executable built from it `ccgen`.

`ccgen` takes a definition file as input and emits C source code that behaves according to that definition. If you feed it `example.def`, it generates `example.c`.

Now suppose we prepare a definition file called `t0.def` and give it to `ccgen`, producing `t4.out.c`. If `t4.out.c` is identical to `t3.out.c`, then `t0.def` defines `ccgen` itself.

The main subject of this document is to understand how that self-defining file `t0.def` works.

# This is a pen

Let us begin with a program that behaves as follows.

- If the input is `This is a pen`, it outputs `I am a boy. F-1 is Formula 1`
- If the input is `It is a ball pencil`, it outputs `It is a ball`

The English sentences themselves do not matter. We are only looking at the shape of the processing.

<table class="plain-table">
  <thead>
    <tr>
      <th>Input file contents</th>
      <th>Output file contents</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td>This is a pen</td>
      <td>I am a boy. F-1 is Formula 1</td>
    </tr>
  </tbody>
</table>

<table class="plain-table">
  <thead>
    <tr>
      <th>Input file contents</th>
      <th>Output file contents</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td>It is a ball pencil</td>
      <td>It is a ball</td>
    </tr>
  </tbody>
</table>


Written directly in C, it would look something like this.

<div class="codeblock-title">sample.c</div>

```c
#include <stdio.h>
#include <unistd.h>
#include <fcntl.h>
int main(int argc, char* argv[])
{
    char buf[256];
    int fd, fd2, ret;
    fd = open(argv[1], O_RDONLY);
    fd2 = open(argv[2], O_WRONLY | O_CREAT);
    ret = read(fd, buf, 256);
    buf[ret] = '\0';
    if (strstr(buf, "This is a pen") != NULL)
        ret = write(fd2, "I am a boy. F-1 is Formula 1", 28);
    else if (strstr(buf, "It is a ball pencil") != NULL)
        write(fd2, "It is a ball", 12);
    else
        printf("syntax error\n");
    close(fd);
    close(fd2);
    return 0;
}
```

`strstr(s1, s2)` checks whether string `s2` occurs inside string `s1`. Error handling is omitted here so we can stay focused on the main topic.

Compile this program into `sample.exe`, and put the following into the input file `pen.txt`.

<table class="plain-table">
  <thead>
    <tr>
      <th>pen.txt (input file)</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td>This is a pen</td>
    </tr>
  </tbody>
</table>

Run it like this.

```bash
$ gcc -o sample.exe sample.c
$ sample.exe pen.txt penout.txt
```

That produces `penout.txt`.

<table class="plain-table">
  <thead>
    <tr>
      <th>penout.txt (output file)</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td>I am a boy. F-1 is Formula 1</td>
    </tr>
  </tbody>
</table>

Up to this point, it is just an ordinary C program.

Next, let us build the same processing with a compiler-compiler.

```bash
# Fetch the source from GitHub
$ git clone https://github.com/t-ishii66/CompilerCompiler.git

# Compile
$ cd CompilerCompiler
$ gcc t3.out.c -Wno-pointer-sign -o ccgen
```

> **Note**: This works on Linux environments such as Ubuntu 24. If you are using Xcode on macOS, install the command line tools first.

`t3.out.c` is the compiler-compiler itself, and `ccgen` is the resulting executable.

Now give `ccgen` a definition file called `PEN.def` that describes logic equivalent to the earlier `sample.c`. For the moment, just look at what `PEN.def` looks like. You do not need to understand it yet.

<div class="codeblock-title">PEN.def</div>

```c
START(PEN0)
PEN0 : THIS " is a " PEN {
                   call(1);
                   print("a");
                   call(2);
                   }
;

THIS : "This" { print("I am "); }
     | "It" { print("It is "); }
;

PEN : "pen" { print(" boy. F-");
              genNum();
              <<
                  print(" is Formula ");
                  genNum();
              >>
            }
    | "ba"+"l"+"l" "pencil" { print(" ball");}
;

END
```

When `PEN.def` is given to `ccgen`, it generates `PEN.c`. Compile that into `PEN.exe`, and you obtain a program that outputs `I am a boy. F-1 is Formula 1` when given `This is a pen`.

```bash
# Compile and run
$ ccgen PEN.def PEN.c
$ gcc -o PEN.exe PEN.c
$ PEN.exe pen.txt penout.txt
```

<table class="plain-table">
  <thead>
    <tr>
      <th>Input file contents (pen.txt)</th>
      <th>Output file contents (penout.txt)</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td>This is a pen</td>
      <td>I am a boy. F-1 is Formula 1</td>
    </tr>
  </tbody>
</table>

<table class="plain-table">
  <thead>
    <tr>
      <th>Input file contents (pen.txt)</th>
      <th>Output file contents (penout.txt)</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td>It is a ball pencil</td>
      <td>It is a ball</td>
    </tr>
  </tbody>
</table>

By this point, it should be clear that the compiler-compiler is a tool that produces C source code from a definition.

# Definition Files

Now let us inspect the contents of `PEN.def`.

Its outer structure is this.

```text
START(PEN0) ... END
```

At the beginning we place `START` and write the language name inside the parentheses. Here that name is `PEN0`. At the end we close with `END`.

Between `START(PEN0)` and `END`, we place definitions of the following form.

```text
meta-variable : definition body
;
```

The left-hand side of `:` is the meta-variable, and the right-hand side is its definition body. A definition ends with `;`. Multiple right-hand sides can be separated by `|`. Here `|` means OR, that is, "or else".

In `PEN.def`, three meta-variables are defined: `PEN0`, `THIS`, and `PEN`.

The basic form is:

```c
meta-variable A : syntax rule { generation rule } ;

meta-variable B : syntax rule 1 { generation rule 1 }
                | syntax rule 2 { generation rule 2 }
                | syntax rule 3 { generation rule 3 }
;
```

Here we treat `syntax rule { generation rule }` as one unit, and for convenience call it a syntax-generation set.

- Syntax rule: a pattern used to match the input string
- Generation rule: processing to run when that syntax rule matches

In the end, a generation rule writes to the current output file, which will be explained later. Multiple syntax-generation sets can be placed side by side with `|`.

For the moment, set the generation rules aside and look only at the syntax rules.

<div class="codeblock-title">PEN.def (syntax rules only)</div>

```c
START(PEN0)
PEN0 : THIS " is a " PEN
;
THIS : "This"
     | "It"
;
PEN : "pen"
    | "ba"+"l"+"l" "pencil"
;
END
```

Writing `START(PEN0)` means that matching against the input begins from `PEN0`.

<div class="codeblock-title">Definition of `PEN0`</div>

```c
PEN0 : THIS " is a " PEN
;
```

On the right-hand side, `" is a "` is a string literal, while `THIS` and `PEN` are meta-variables.

So `PEN0` accepts input of the form "pattern defined by `THIS` + string `" is a "` + pattern defined by `PEN`".

Next, `THIS`.

```c
THIS : "This"
     | "It"
;
```

Because of `|`, this matches either `This` or `It`. So at this stage `PEN0` accepts input that begins with either `This is a ...` or `It is a ...`.

Now look at `PEN`.

```c
PEN : "pen"
    | "ba"+"l"+"l" "pencil"
;
```

The alternatives here are `pen`, or `ba+l+l pencil`.

`+` means "do not allow spaces between these strings". For example, `"ba"+"l"` fails if there is whitespace between `ba` and `l`.

To show the difference between `"ba" "l"` and `"ba"+"l"`:

```text
bal
ba l
ba     l
```

`"ba" "l"` matches all of those.

By contrast, `"ba"+"l"` matches only `bal`.

So `"ba"+"l"+"l"` is effectively the same as `"ball"`, but it is split this way on purpose here so the role of `+` is visible.

Also, `"ball" "pencil"` allows there to be a space between the two words, or not. But whitespace cannot appear in the middle of either word. By contrast, if you write `"ball pencil"` as one string literal, then exactly one space is required.

From this, `PEN0` can process inputs such as `This is a pen` or `It is a ball pencil`. Looking only at the syntax, the idea is the same as in the first `sample.c`.

## Generation Rules

Next come the generation rules. Start with `PEN0`.

<div class="codeblock-title">Syntax rule and generation rule of `PEN0`</div>

```c
PEN0 : THIS " is a " PEN {
    call(1);
    print("a");
    call(2);
}
;
```

`call(1);` corresponds to `THIS`, and `call(2);` corresponds to `PEN`. `call(n)` means "execute the generation rule of the corresponding meta-variable", where "corresponding" means the nth meta-variable that appeared.

The `print("a");` in the middle writes the string `a` to the output file.

So the generation rule of `PEN0` behaves in this order:

```text
1. Execute the generation rule of THIS
2. Output "a"
3. Execute the generation rule of PEN
```

For `THIS`, we have:

```c
THIS : "This" { print("I am "); }
     | "It" { print("It is "); }
;
```

If the input is `This`, it outputs `I am `. If the input is `It`, it outputs `It is `.

Finally, look at `PEN`.

```c
PEN : "pen" { print(" boy. F-");
              genNum();
              <<
                  print(" is Formula ");
                  genNum();
              >>
            }
    | "ba"+"l"+"l" "pencil" { print(" ball");}
;
```

`PEN` has two syntax-generation sets.

If the input is `It is a ball pencil`, the prefix `It is a ` has already been handled by the first half of `PEN0`. The remaining `ball pencil` matches `"ba"+"l"+"l" "pencil"` in `PEN`, and then the generation rule `print(" ball");` runs. So the final output is `It is a ball`.

Now consider the case where the input is `This is a pen`. Then the generation rule on the `PEN` side is:

```c
print(" boy. F-");
genNum();
<<
    print(" is Formula ");
    genNum();
>>
```

First, `print(" boy. F-");` outputs ` boy. F-`.

`genNum();` is an instruction that generates a numeric string. Within the same generation rule, the same number is reused. So the two occurrences of `genNum();` in this `PEN` rule produce the same value.

The initial value is 1, so the first `genNum();` used in this rule produces 1. If another rule uses it, that rule gets the next number.

In effect, this behaves like:

```c
print(" boy. F-");
print("1");
<<
    print(" is Formula ");
    print("1");
>>
```

The final `<< ... >>` is a mechanism that appends the output produced inside it to the end of the final output file.

In this example, ` boy. F-1` is produced first, and then ` is Formula 1` is appended at the end.

Putting everything together, the whole behavior is:

<table class="plain-table">
  <thead>
    <tr>
      <th>Input file contents</th>
      <th>Output file contents</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td>This is a pen</td>
      <td>I am a boy. F-1 is Formula 1</td>
    </tr>
    <tr>
      <td>It is a ball pencil</td>
      <td>It is a ball</td>
    </tr>
  </tbody>
</table>

Let us confirm this on the actual implementation.

```bash
$ ./ccgen PEN.def PEN.c
```

This generates `PEN.c`. `PEN.c` is a C program that parses the input string and outputs the result.

Since `ccgen` generates that `PEN.c`, it is called a compiler-compiler: a compiler that creates compilers.

<table class="plain-table">
<colgroup>
<col style="width: 49%" />
<col style="width: 50%" />
</colgroup>
<tbody>
<tr>
<td>PEN.def</td>
<td>PEN.c</td>
</tr>
<tr>
<td>

```c
START(PEN0)
PEN0 : THIS " is a " PEN {
                         call(1);
                         print("a");
                         call(2);
                         }
;
THIS : "This" {print("I am ");}
     | "It" {print("It is ");}
;
PEN : "pen" {print(" boy. F-");
             genNum();
             <<
                print(" is Formula ");
                genNum();
             >>
            }
    | "ba"+"l"+"l" "pencil"
            { print(" ball");}
;
END
```

</td>
<td>

```c
#include "comcom.h"
void firstcall(int e) { _PEN0(e); }

void _PEN0(int e)
{
    _THIS(E);
    check(" is a ");
    _PEN(E);
    mknode(_f1);
}

void _THIS(int e)
{
    if (try()) {
        check("This");
        mknode(_f2);
        ok();
    } else {
        check("It");
        mknode(_f3);
    }
}

void _PEN(int e)
{
    if (try()) {
        check("pen");
        mknode(_f4);
        ok();
    } else {
        check("ba");
        combine();
        check("l");
        combine();
        check("l");
        check("pencil");
        mknode(_f5);
    }
}

void _f1(int n, int d)
{
    call(1);
    print("a");
    call(2);
}

void _f2(int n, int d) { print("I am "); }

void _f3(int n, int d) { print("It is "); }

void _f4(int n, int d)
{
    print(" boy. F-");
    genNum();
    spoolOpen();
    print(" is Formula ");
    genNum();
    spoolClose();
}

void _f5(int n, int d) { print(" ball"); }
```

</td>
</tr>
</tbody>
</table>

Compile `PEN.c` to build `PEN.exe`.

```bash
$ gcc -o PEN.exe PEN.c
```

Then verify that it behaves the same way as `sample.exe`. Let the input file be `pen.txt` and the output file be `penout2.txt`.

```bash
$ ./PEN.exe pen.txt penout2.txt
```

If the expected output appears, then the `PEN.c` generated from `PEN.def` is performing the intended processing.

But what we really want to understand is why it works that way. Looking only at the result, `sample.c` and `PEN.c` do almost the same job, but the inside of `PEN.c` is structured in a way that is difficult to read directly.

To understand the compiler-compiler, we need to trace what this generated `PEN.c` is doing. The next chapter examines that internal structure in detail.
