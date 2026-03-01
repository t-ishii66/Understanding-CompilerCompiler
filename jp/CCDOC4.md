# PEN.c の大局的理解

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

ここまでの内容を使えば、`ccgen` が `PEN.def` から生成した `PEN.c` を、細部に立ち入りすぎずに読めるはずです。大切なのは、局所的な構文よりも全体の流れをつかむことです。

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

このコードを見て、入力が `This is a pen` のときに `I am a boy. F-1 is Formula 1` が出力され、入力が `It is a ball pencil` のときに `It is a ball` が出力される理由が追えるようになれば、この章の前半は読み進められると思います。

# 自己定義

## 立ち位置

ここからの目標は、`PEN.def` を `PEN.c` に変換する規則そのものを明らかにすることです。つまり、`ccgen` が何をしているのかを文法として記述したいわけです。

その規則を `t0.def` として書ければ、次のコマンドで `ccgen` 自身に相当する C ソースが得られます。

```bash
$ ccgen t0.def t4.out.c
```

もし `t4.out.c` が `t3.out.c`（ccgenのソースコード） と一致するなら、`t0.def` は `ccgen` の自己定義と言えます。

まずは、`PEN.def` と `PEN.c` の対応をもう一度並べます。

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

ここで立場を混乱させないことが重要です。今の立場を表にすると次の通りです。

<table class="plain-table">
  <thead>
    <tr>
      <th>定義ファイル（ccgen の入力）</th>
      <th>生成ファイル（ccgen が生成する C ソース）</th>
      <th>入力ファイル（左隣の C コードへの入力）</th>
      <th>出力ファイル（２つ左隣の C コードの出力）</th>
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

この関係が出発点です。`t0.def` は、`PEN.def` を入力として `PEN.c` を出力する規則でなければなりません。

入力となる `PEN.def` は、たとえば次のような形を持っています。

<div class="codeblock-title">PEN.def（入力ファイル）</div>

```c
START(PEN0)
...
END
```

これに対し、出力される `PEN.c` の先頭は次の通りです。

<div class="codeblock-title">PEN.c（出力ファイル）</div>

```c
#include "comcom.h"
void firstcall(int e) { _PEN0(e); }
```

したがって、`t0.def` の入口はたとえば次のように考えられます。

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

この時点では、形だけ合っていれば十分です。

処理すべき `PEN.def` の本体は次のような並びです。

<div class="codeblock-title">PEN.def（一部）</div>

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

まず最初の 1 行だけをそのまま処理しようとすると、`STMTS` は次のような非常に具体的な規則になります。

<div class="codeblock-title">t0.def（一部）</div>

```c
STMTS : "PEN0" ":" "THIS" "\" is a \"" "PEN" "{"
    "call(1);" "print(\"a\");" "call(2);" "}" {...}
;
```

もちろん、これでは一般化できません。ここから抽象化が必要になります。

ここで大事なのは、「`PEN.def` の中にある記号（構文規則だろうが生成規則だろうが）を、`t0.def` から見るとただの入力文字列としてマッチさせる（つまり、構文規則として）」という視点です。

- `PEN.def` に `THIS` と書かれていたら、`t0.def` では `"THIS"` と書いてマッチさせる
- `PEN.def` に `" is a "` と書かれていたら、`t0.def` では `"\" is a \""` と書いてマッチさせる
- `PEN.def` に `+` と書かれていたら、`t0.def` では `"+"` と書いてマッチさせる

## 再帰構造

まずは `STMTS` を「複数の定義文の列」として扱えるようにします。

<div class="codeblock-title">t0.def（一部）</div>

```c
STMTS : STMT STMTS { call(1); call(2); }
| STMT { call(1); }
;

STMT : NAME ":" RULES ";" { ... }
;
```

これは典型的な再帰構造です。例として、次の `SS.def` を考えます。

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

`SS` は「`STM` だけ」または「`STM` の後にさらに `SS` が続くもの」です。したがって、この文法が受理する入力は次のような形になります。

<table class="plain-table">
  <thead>
    <tr>
      <th>サンプルSSが期待する入力</th>
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

`ccgen` が生成した `SS.c` は次のようになります。

<div class="codeblock-title">SS.c（ccgen により生成）</div>

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

入力が `This This This` のような場合、解析木 `TS[]` はバックトラックを伴いながら構築されます。細部は追えるなら追えばよいですが、ここで大切なのは、再帰構造と `try()/back()/ok()` によって「長い列」が自然に表現できることです。

次に、`STMTS` の生成規則を考えます。`STMTS` は複数の `STMT` から成り、その生成規則も順番どおりに呼ばれてほしいので、

```c
STMTS : STMT STMTS { call(1); call(2); }
| STMT { call(1); }
;
```

という形はそのままでよさそうです。

次は `STMT` の先頭にある `NAME` です。

## 超変数 NAME

`NAME` は次のように定義できます。

<div class="codeblock-title">t0.def（一部）</div>

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

再帰構造なので、`NAME` は「先頭が英字で、その後に英字や数字が続く文字列」を表します。これで `PEN0`, `THIS`, `PEN` といった名前にマッチできます。

生成規則も重要で、ここではマッチした名前をそのまま出力しています。

## 構文生成セット

`STMT` の次の要素は `RULES` です。

<div class="codeblock-title">t0.def（一部）</div>

```c
STMT : NAME ":" RULES ";" { ... }
;
```

`RULES` は複数の構文生成セットの列として考えます。

<div class="codeblock-title">t0.def（一部）</div>

```c
RULES : RULE "|" RULES {...}
      | RULE {...}
;
```

ここでの `"|"` は、入力ファイル `PEN.def` に書かれている縦棒という文字にマッチさせるためのパターンです。`t0.def` 自体の文法で使う `|` とは別物です。

`PEN.def` で見ると、`THIS` と `PEN` は複数の `RULE` から構成されています。

1 つの `RULE` は次の形です。

| `構文規則 { 生成規則 }` |
|---|

これを素直に表すと、`RULE` は次になります。

<div class="codeblock-title">t0.def（一部）</div>

```c
RULE : TERMS "{" OUTS "}" {...}
;
```

ここで `TERMS` は構文規則部分、`OUTS` は生成規則部分です。

`TERMS` に現れる例は `PEN.def` では次の通りです。

<div class="codeblock-title">PEN.def で登場するパターン（TERMS としてマッチさせるもの）</div>

```text
THIS " is a " PEN
"This"
"It"
"pen"
"ba"+"l"+"l" "pencil"
```

ここでは最小単位 `TERM` を導入し、`TERMS` はその列とします。

<div class="codeblock-title">t0.def（一部）</div>

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

ここでの対応関係は次のようになります。

<table class="plain-table">
  <thead>
    <tr>
      <th>入力パターン（PEN.def）</th>
      <th>t0.def の TERM でマッチするもの</th>
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

この定義により、`PEN.def` の構文規則部分は次のような C コードに変換されます。

<table class="plain-table">
  <thead>
    <tr>
      <th>入力パターン</th>
      <th>出力パターン</th>
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

`PEN.c` を見返すと、確かにその形になっています。

<div class="codeblock-title">PEN.c（一部）</div>

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

ここでまだ残るのは、最後に必ず付く `mknode(_f数字);` の部分です。したがって `RULE` は次のように考えられます。

<div class="codeblock-title">t0.def（一部）</div>

```c
RULE : TERMS "{" OUTS "}" { call(1); print(" mknode(_f");
                            genNum();
                            print(");");
                            ...
                          }
;
```

`call(1);` で `TERMS` 側の出力を行い、その後で `mknode(_f数字);` を付ければよさそうです。

## 生成規則

次は `RULE` の後半、つまり生成規則側の `OUTS` を考えます。

`PEN.def` の生成規則部分を抜き出すと、次のようなパターンが現れます。

<div class="codeblock-title">PEN.def で登場する生成規則パターン</div>

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

ここでも `OUTS` を `OUT` の列として定義します。

<div class="codeblock-title">t0.def（一部）</div>

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

ここでも、`NAME` が無いことに注意してください。生成規則部分には超変数名が現れないからです。

`OUT` の規則は、基本的に入力された生成規則をそのまま C コードとして出力する形になっています。例を見ます。

<table class="plain-table">
  <thead>
    <tr>
      <th>入力パターン（PEN.def の生成規則）</th>
      <th>t0.defのOUT でマッチする規則</th>
      <th>出力</th>
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

つまり、ここでは「同じものを出力する」ように定義しているわけです。

`<< ... >>` の場合だけが例外で、この部分は `spoolOpen();` と `spoolClose();` で囲まれます。

たとえば次の入力は、

<div class="codeblock-title">入力データ</div>

```c
<<
print(" is Formula ");
genNum();
>>
```

出力では次の形になります。

<div class="codeblock-title">出力されるコード</div>

```c
spoolOpen();
print(" is Formula ");
genNum();
spoolClose();
```

したがって、`OUTS` の中身そのものは出力できそうです。

さて問題は、`TERMS`にマッチしたら、mknodeで`TS[]`に積まれた `_f数字()` を後で呼び出すことになりますがその実体、`void _f数字(int n, int d) { ... }` をどこで出力するかです。これは `RULE` で受け持つのが自然です。

<div class="codeblock-title">t0.def（一部）</div>

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

ここで `genNum();` を 2 回使っていることが重要です。同じ生成規則の中なので、この 2 つは同じ番号になります。つまり、たとえば `mknode(_f2);` と出したなら、あとで `void _f2(...) { ... }` も同じ番号になります。

したがって、最終的には次のような対応になります。

```c
_THIS(e) { .... ; mknode(_f2); }
...
_f2(int n, int d) {
    ...
}
```

これは `PEN.c` の構造と一致しています。

## 選択規則

最後に残るのが `|` です。

`PEN.def` では、複数の構文生成セットを `|` で並べられます。

<div class="codeblock-title">PEN.def（一部）</div>

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

これを `RULES` として表すなら、次のように定義できます。

<div class="codeblock-title">t0.def（一部）</div>

```c
RULES : RULE "|" RULES { print(" if (try()) {"); call(1);
                         print(" ok(); }\n else "); call(2); }
      | RULE { print("{"); call(1); print("}"); }
;
```

ここでの狙いは、`ccgen` が `|` を `if (try()) { ... } else { ... }` へ変換しているのと同じことを、`t0.def` でも再現することです。

この規則により、たとえば `PEN.def` の `PEN` の定義は、`PEN.c` では次のように出力されます。

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


これで、`PEN.def` を `PEN.c` に変換するための主要な規則が揃いました。

ここまでの議論では、`PEN.def` に特有の記述だけでなく、できるだけ一般の入力にも通用するように `t0.def` を組み立ててきました。その結果、`t0.def` が扱える文法は、実質的に `ccgen` 自身が扱える文法と同じところまで到達しています。

つまり、当初は `PEN.def` を `PEN.c` に変換する規則として作り始めた `t0.def` が、そのまま `ccgen` 自身の定義になっているわけです。

完成した `t0.def` から C ソースを生成するには、次を実行します。

```bash
$ ccgen t0.def t4.out.c
```

それをコンパイルすれば、`ccgen` と同じ機能を持つ新しいコンパイラコンパイラが得られます。

```bash
$ gcc -o new-ccgen t4.out.c
```

これで、この段階の目的は達成です。
