# A Macroscopic Understanding of `PEN.c`

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

With everything covered so far, you should now be able to read the `PEN.c` that `ccgen` generated from `PEN.def` without getting lost in the details. The important thing is not local syntax, but the overall flow.

<div class="codeblock-title">PEN.c</div>

```c
#include "comcom.h"
void firstcall(int e) { _PEN0(e); }
void _PEN0(int e) {
    { _THIS(E); check(" is a "); _PEN(E); mknode(_f1);}
}
void _THIS(int e) {
    if (try()) { check("This"); mknode(_f2); ok(); }
    else { check("It"); mknode(_f3);}
}
void _PEN(int e) {
    if (try()) { check("pen"); mknode(_f4); ok(); }
    else { check("ba"); combine(); check("l"); combine(); check("l");
check("pencil"); mknode(_f5);}
}
void _f1(int n, int d) {call(1);print("a");call(2);}
void _f2(int n, int d) {print("I am ");}
void _f3(int n, int d) {print("It is ");}
void _f4(int n, int d) {print(" boy. F-");genNum();spoolOpen();
print(" is Formula ");genNum();spoolClose();
}
void _f5(int n, int d) {print(" ball");}
```

If, by reading this code, you can follow why the input `This is a pen` produces `I am a boy. F-1 is Formula 1`, and why the input `It is a ball pencil` produces `It is a ball`, then the first half of this chapter has served its purpose.

# Self-Definition

## Where We Stand

From here on, the goal is to make explicit the rules that turn `PEN.def` into `PEN.c`. In other words, we want to describe, as a grammar, what `ccgen` itself is doing.

If we can write those rules as `t0.def`, then the following command will produce C source equivalent to `ccgen` itself.

```bash
$ ccgen t0.def t4.out.c
```

If `t4.out.c` is identical to `t3.out.c` (the source code of `ccgen`), then `t0.def` can be called a self-definition of `ccgen`.

First, place `PEN.def` and `PEN.c` side by side once more.

<table class="plain-table">
<colgroup>
<col style="width: 50%" />
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
THIS : "This" { print("I am "); }
| "It"    { print("It is "); }
;
PEN : "pen"{
        print(" boy. F-");
        genNum();
        <<
            print(" is Formula ");
            genNum();
        >>
    }
| "ba"+"l"+"l" "pencil" {
        print(" ball"); }
;
END
```
</td>
<td>

```c
#include "comcom.h"

void firstcall(int e) { _PEN0(e); }

void _PEN0(int e) {

{ _THIS(E); check(" is a "); _PEN(E); mknode(_f1);}

}

void _THIS(int e) {

if (try()) { check("This"); mknode(_f2); ok(); }

else { check("It"); mknode(_f3);}

}

void _PEN(int e) {

if (try()) { check("pen"); mknode(_f4); ok(); }

else { check("ba"); combine(); check("l"); combine(); check("l"); check("pencil"); mknode(_f5);}

}

void _f1(int n, int d) {call(1);print("a");call(2);}

void _f2(int n, int d) {print("I am ");}

void _f3(int n, int d) {print("It is ");}

void _f4(int n, int d) {print(" boy. F-");genNum();spoolOpen();

print(" is Formula ");genNum();spoolClose();

}

void _f5(int n, int d) {print(" ball");}
```

</td>
</tr>
</tbody>
</table>

It is important not to confuse levels here. The current standpoint looks like this:

<table class="plain-table">
  <thead>
    <tr>
      <th>Definition file (input to ccgen)</th>
      <th>Generated file (C source emitted by ccgen)</th>
      <th>Input file to the C code on its left</th>
      <th>Output file of the C code two steps to its left</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td>PEN.def</td>
      <td>PEN.c</td>
      <td>pen.txt</td>
      <td>penout.txt</td>
    </tr>
    <tr>
      <td>t0.def</td>
      <td>t4.out.c</td>
      <td>PEN.def</td>
      <td>PEN.c</td>
    </tr>
  </tbody>
</table>

This relation is the starting point. `t0.def` must be a set of rules that takes `PEN.def` as input and outputs `PEN.c`.

The input `PEN.def` has a shape like this:

<div class="codeblock-title">PEN.def (input file)</div>

```c
START(PEN0)
...
END
```

Against that, the beginning of the output `PEN.c` is:

<div class="codeblock-title">PEN.c (output file)</div>

```c
#include "comcom.h"
void firstcall(int e) { _PEN0(e); }
```

So the entry of `t0.def` might begin like this:

<div class="codeblock-title">t0.def</div>

```c
START(COMCOM)
COMCOM : "START(" NAME ")" STMTS "END" {
    print("#include \"comcom.h\"\n");
    <<
    print("void firstcall(int e) { _");
    call(1);
    print("(e); }\n");
    >>
    call(2); }
STMTS : ...
;
END
```

At this stage, getting the shape right is enough.

The body of `PEN.def` that must be processed looks like this:

<div class="codeblock-title">PEN.def (excerpt)</div>

```c
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
    | "ba"+"l"+"l" "pencil" { print(" ball"); }
;
```

If you tried to process only the first line literally, `STMTS` would become an absurdly specific rule like this:

<div class="codeblock-title">t0.def (excerpt)</div>

```c
STMTS : "PEN0" ":" "THIS" "\" is a \"" "PEN" "{"
    "call(1);" "print(\"a\");" "call(2);" "}" {...}
;
```

Obviously that cannot be generalized. So abstraction is required.

The key viewpoint is this: from the standpoint of `t0.def`, every symbol inside `PEN.def`, whether it belongs to a syntax rule or a generation rule, is merely an input character sequence to be matched as syntax.

- If `PEN.def` contains `THIS`, then `t0.def` matches it by writing `"THIS"`
- If `PEN.def` contains `" is a "`, then `t0.def` matches it by writing `"\" is a \""`
- If `PEN.def` contains `+`, then `t0.def` matches it by writing `"+"`

## Recursive Structure

First, let us make `STMTS` capable of representing a sequence of multiple definition statements.

<div class="codeblock-title">t0.def (excerpt)</div>

```c
STMTS : STMT STMTS { call(1); call(2); }
| STMT { call(1); }
;

STMT : NAME ":" RULES ";" { ... }
;
```

This is a standard recursive structure. As a simple example, consider the following `SS.def`.

<div class="codeblock-title">SS.def</div>

```c
START(SS)

SS : STM SS { call(1); call(2); }
| STM { call(1); }
;

STM : "This" { print("T"); }
;
END
```

`SS` means either "just `STM`" or "an `STM` followed by another `SS`". Therefore this grammar accepts inputs like:

<table class="plain-table">
  <thead>
    <tr>
      <th>Input expected by the sample SS grammar</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td>This</td>
    </tr>
    <tr>
      <td>This This</td>
    </tr>
    <tr>
      <td>This ThisThisThis This</td>
    </tr>
  </tbody>
</table>

The `SS.c` generated by `ccgen` looks like this:

<div class="codeblock-title">`SS.c` (generated by `ccgen`)</div>

```c
#include "comcom.h"
void firstcall(int e) { _SS(e); }
void _SS(int e) {
    if (try()) { _STM(E); _SS(E); mknode(_f1); ok(); }
    else { _STM(E); mknode(_f2);}
}
void _STM(int e) {
    { check("This"); mknode(_f3);}
}
void _f1(int n, int d) {call(1);call(2);}
void _f2(int n, int d) {call(1);}
void _f3(int n, int d) {print("T");}
```

When the input is something like `This This This`, the parse tree `TS[]` is built while backtracking occurs as needed. You can trace the details if you want, but the important point here is simply that recursive structure, together with `try()/back()/ok()`, naturally expresses a long sequence.

Next, think about the generation rule for `STMTS`. Since `STMTS` consists of multiple `STMT`s and we want their generation rules to run in order, this shape still seems right:

```c
STMTS : STMT STMTS { call(1); call(2); }
| STMT { call(1); }
;
```

The next element at the start of `STMT` is `NAME`.

## Meta-Variable `NAME`

`NAME` can be defined like this:

<div class="codeblock-title">t0.def (excerpt)</div>

```c
NAME : CHR+NAME { call(1); call(2); }
     | CHR { call(1); }
;
CHR : ALPH { call(1); }
    | SALPH { call(1); }
    | NUM { call(1); }
;
NUMS : NUM+NUMS { call(1); call(2); }
     | NUM { call(1); }
;
ALPH : "A" { print("A"); }
     ...
     | "Z" { print("Z"); }
;
SALPH : "a" { print("a"); }
      ...
      | "z" { print("z"); }
;
NUM : "0" { print("0"); }
    ...
    | "9" { print("9"); }
;
```

Because of the recursive structure, `NAME` denotes a string that starts with a letter and is followed by letters or digits. That allows it to match names such as `PEN0`, `THIS`, and `PEN`.

The generation rules matter too. Here they simply output the matched name as-is.

## Syntax-Generation Sets

The next piece in `STMT` is `RULES`.

<div class="codeblock-title">t0.def (excerpt)</div>

```c
STMT : NAME ":" RULES ";" { ... }
;
```

We can think of `RULES` as a sequence of syntax-generation sets.

<div class="codeblock-title">t0.def (excerpt)</div>

```c
RULES : RULE "|" RULES {...}
      | RULE {...}
;
```

The `"|"` here is a pattern that matches the literal vertical bar character written in the input file `PEN.def`. It is different from the `|` used by the grammar of `t0.def` itself.

In `PEN.def`, both `THIS` and `PEN` are made of multiple `RULE`s.

One `RULE` has the following form:

| syntax rule { generation rule } |
|---|

So, in a straightforward form:

<div class="codeblock-title">t0.def (excerpt)</div>

```c
RULE : TERMS "{" OUTS "}" {...}
;
```

Here `TERMS` is the syntax-rule part, and `OUTS` is the generation-rule part.

The examples of `TERMS` appearing in `PEN.def` are:

<div class="codeblock-title">Patterns appearing in `PEN.def` (things to match as `TERMS`)</div>

```text
THIS " is a " PEN
"This"
"It"
"pen"
"ba"+"l"+"l" "pencil"
```

Introduce the minimum unit `TERM`, and let `TERMS` be a sequence of them.

<div class="codeblock-title">t0.def (excerpt)</div>

```text
TERMS : TERM TERMS { call(1); call(2); }
      | TERM { call(1); }
;
TERM : NAME { print(" _"); call(1);
              print("(E);"); }
     | QSTR { print(" check("); call(1);
              print(");"); }
     | "+" { print(" combine();"); }
;
```

The correspondence here is:

<table class="plain-table">
  <thead>
    <tr>
      <th>Input pattern (PEN.def)</th>
      <th>What matches it in TERM of t0.def</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td>THIS, PEN</td>
      <td>NAME</td>
    </tr>
    <tr>
      <td>" is a ", "This", "pen", "ba", "l", "pencil"</td>
      <td>QSTR</td>
    </tr>
    <tr>
      <td>+</td>
      <td>"+"</td>
    </tr>
  </tbody>
</table>

With this definition, the syntax-rule part of `PEN.def` is converted into the following C code.

<table class="plain-table">
  <thead>
    <tr>
      <th>Input pattern</th>
      <th>Output pattern</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td>THIS " is a " PEN</td>
      <td>_THIS(E);check(" is a ");_PEN(E);</td>
    </tr>
    <tr>
      <td>"This"</td>
      <td>check("This");</td>
    </tr>
    <tr>
      <td>"It"</td>
      <td>check("It");</td>
    </tr>
    <tr>
      <td>"pen"</td>
      <td>check("pen");</td>
    </tr>
    <tr>
      <td>"ba"+"l"+"l" "pencil"</td>
      <td>check("ba");combine();check("l");combine();check("l");check("pencil");</td>
    </tr>
  </tbody>
</table>

If you look back at `PEN.c`, it is indeed in that form.

<div class="codeblock-title">PEN.c (excerpt)</div>

```c
void _PEN0(int e) {
    { _THIS(E); check(" is a "); _PEN(E); mknode(_f1);}
}
void _THIS(int e) {
    if (try()) { check("This"); mknode(_f2); ok(); }
    else { check("It"); mknode(_f3);}
}
void _PEN(int e) {
    if (try()) { check("pen"); mknode(_f4); ok(); }
    else { check("ba"); combine(); check("l"); combine(); check("l");
check("pencil"); mknode(_f5);}
}
```

What is still missing here is the `mknode(_f<number>);` that always appears at the end. So `RULE` should probably look like this:

<div class="codeblock-title">t0.def (excerpt)</div>

```c
RULE : TERMS "{" OUTS "}" { call(1); print(" mknode(_f");
                            genNum();
                            print(");");
                            ...
                          }
;
```

If `call(1);` outputs the code corresponding to `TERMS`, and then `mknode(_f<number>);` is appended, that part seems accounted for.

## Generation Rules

Next is the second half of `RULE`, namely the `OUTS` on the generation-rule side.

If we extract the generation-rule patterns that appear in `PEN.def`, we get:

<div class="codeblock-title">Generation-rule patterns appearing in `PEN.def`</div>

```text
call(1);
print("a");
call(2);
print("I am ");
print("It is ");
print(" boy. F-");
genNum();
<< print(" is Formula "); genNum(); >>
print(" ball");
```

Again, define `OUTS` as a sequence of `OUT`.

<div class="codeblock-title">t0.def (excerpt)</div>

```c
OUTS : OUT OUTS { call(1); call(2); }
     | OUT { call(1); }
;
OUT : "call(" NUMS ");" { print("call("); call(1);
                          print(");"); }
    | "print(" QSTR ");" { print("print("); call(1);
                           print(");"); }
    | "genNum();" { print("genNum();"); }
    | "<<" OUTS ">>" { print("spoolOpen();\n"); call(1);
                       print("spoolClose();\n"); }
;
```

Notice that there is no `NAME` here. Meta-variable names do not appear in generation-rule parts.

Each rule for `OUT` basically outputs the input generation rule unchanged as C code. For example:

<table class="plain-table">
  <thead>
    <tr>
      <th>Input pattern (generation rule in PEN.def)</th>
      <th>Rule in OUT of t0.def that matches it</th>
      <th>Output</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td>call(1);</td>
      <td>"call(" NUMS ");"</td>
      <td>call(1);</td>
    </tr>
    <tr>
      <td>print(" is not ");</td>
      <td>"print(" QSTR ");"</td>
      <td>print(" is not ");</td>
    </tr>
    <tr>
      <td>genNum();</td>
      <td>"genNum();"</td>
      <td>genNum();</td>
    </tr>
  </tbody>
</table>

So the definition here is simply "output the same thing".

The only exception is `<< ... >>`, which is wrapped with `spoolOpen();` and `spoolClose();`.

For example, if the input is:

<div class="codeblock-title">Input</div>

```c
<<
print(" is Formula ");
genNum();
>>
```

then the output becomes:

<div class="codeblock-title">Generated code</div>

```c
spoolOpen();
print(" is Formula ");
genNum();
spoolClose();
```

So the contents of `OUTS` itself seem producible.

The remaining problem is this: when `TERMS` matches, a corresponding `_f<number>()` is pushed into `TS[]` by `mknode`, and that function body must be emitted somewhere later as `void _f<number>(int n, int d) { ... }`. The natural place to handle this is `RULE`.

<div class="codeblock-title">t0.def (excerpt)</div>

```text
RULE : TERMS "{" OUTS "}" { call(1); print(" mknode(_f");
                            genNum();
                            print(");");
                            <<
                            print("void _f");
                            genNum();
                            print("(int n, int a) {");
                            call(2);
                            print("}\n");
                            >>
                          }
;
```

The important point is that `genNum();` appears twice here. Because they occur within the same generation rule, they yield the same number. So if the first one outputs `mknode(_f2);`, the second one later outputs `void _f2(...) { ... }`.

That gives the final correspondence:

```c
_THIS(e) { .... ; mknode(_f2); }
...
_f2(int n, int d) {
    ...
}
```

This matches the structure of `PEN.c`.

## Selection Rules

The last missing piece is `|`.

In `PEN.def`, multiple syntax-generation sets can be listed using `|`.

<div class="codeblock-title">PEN.def (excerpt)</div>

```c
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
```

If we represent that as `RULES`, we can define:

<div class="codeblock-title">t0.def (excerpt)</div>

```c
RULES : RULE "|" RULES { print(" if (try()) {"); call(1);
                         print(" ok(); }\n else "); call(2); }
      | RULE { print("{"); call(1); print("}"); }
;
```

The point is to reproduce in `t0.def` the same conversion that `ccgen` performs, namely turning `|` into `if (try()) { ... } else { ... }`.

With this rule, for example, the definition of `PEN` in `PEN.def` is emitted into `PEN.c` like this:

<table class="plain-table">
<colgroup>
<col style="width: 50%" />
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
PEN : "pen" ...
    | "ba"+"l"...
;
```
</td>
<td>

```c
void _PEN(int e) {

if (try()) { check("pen"); mknode(_f4); ok(); }

else { check("ba"); combine(); check("l"); combine(); check("l"); check("pencil"); mknode(_f5);}

}
```
</td>
</tr>
</tbody>
</table>

At this point, the main rules needed to convert `PEN.def` into `PEN.c` are all in place.

In the discussion so far, `t0.def` was assembled so that it would work not only for notation specific to `PEN.def`, but for as much general input as possible. As a result, the grammar handled by `t0.def` has effectively reached the same level as the grammar handled by `ccgen` itself.

That means the `t0.def` we began by constructing as "rules for converting `PEN.def` into `PEN.c`" has, in the end, become the definition of `ccgen` itself.

To generate C source from the completed `t0.def`, run:

```bash
$ ccgen t0.def t4.out.c
```

Compile it, and you obtain a new compiler-compiler with the same functionality as `ccgen`.

```bash
$ gcc -o new-ccgen t4.out.c
```

At that point, the goal of this stage has been achieved.
