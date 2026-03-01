# コンパイラコンパイラ from スクラッチ

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

「コンパイラコンパイラ」は誤植ではありません。コンパイラを生成する処理系を指す言葉で、yacc や bison が代表例です。

本書で扱うのは、小さなコンパイラコンパイラを自作し、そのうえでその処理系自身を自分の文法で定義する、という自己定義の仕組みです。

最初は少し回りくどく見えるかもしれませんが、具体例を順に追えば全体像は見えてきます。ここでは細部に立ち止まりすぎず、まずは流れをつかむことを優先します。

# 目的

本書で扱うコンパイラコンパイラは C 言語で書かれています。ソースファイルは `t3.out.c`、その実行ファイルを `ccgen` とします。

`ccgen` は、定義ファイルを入力すると、その定義に従って動作する C ソースを出力します。たとえば `example.def` を入力すれば `example.c` が生成されます。

ここで `t0.def` という定義ファイルを用意し、これを `ccgen` に与えて `t4.out.c` を生成します。もし `t4.out.c` が `t3.out.c` と一致すれば、`t0.def` は `ccgen` 自身を定義していることになります。

本書の主題は、この自己定義ファイル `t0.def` の仕組みを理解することです。

# This is a pen

まずは、次のように振る舞うプログラムを考えます。

- 入力が `This is a pen` なら `I am a boy. F-1 is Formula 1` を出力する
- 入力が `It is a ball pencil` なら `It is a ball` を出力する

英文そのものに意味はありません。ここでは処理の形だけを見ます。

<table class="plain-table">
  <thead>
    <tr>
      <th>入力ファイルの中身</th>
      <th>出力ファイルの中身</th>
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
      <th>入力ファイルの中身</th>
      <th>出力ファイルの中身</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td>It is a ball pencil</td>
      <td>It is a ball</td>
    </tr>
  </tbody>
</table>


これを C で素直に書くと、たとえば次のようになります。

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

`strstr(s1, s2)` は、文字列 `s1` の中に `s2` が含まれているかを調べる関数です。ここでは本題に集中するため、エラーチェックは省略しています。

このプログラムをコンパイルして `sample.exe` を作り、入力ファイル `pen.txt` に次を書きます。

<table class="plain-table">
  <thead>
    <tr>
      <th>pen.txt（入力ファイル）</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td>This is a pen</td>
    </tr>
  </tbody>
</table>

実行は次の通りです。

```bash
$ gcc -o sample.exe sample.c
$ sample.exe pen.txt penout.txt
```

すると `penout.txt` が生成されます。

<table class="plain-table">
  <thead>
    <tr>
      <th>penout.txt（出力ファイル）</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td>I am a boy. F-1 is Formula 1</td>
    </tr>
  </tbody>
</table>

ここまでは普通の C プログラムです。

次に、同じ処理をコンパイラコンパイラで作ります。

```bash
# GitHub からソースを取得
$ git clone https://github.com/t-ishii66/CompilerCompiler.git

# コンパイル
$ cd CompilerCompiler
$ gcc t3.out.c -Wno-pointer-sign -o ccgen
```

> **注意**: Ubuntu 24 などの Linux 環境で動作します。macOS の Xcode を使う場合は、コマンドラインツールズをインストールしてください。

`t3.out.c` がコンパイラコンパイラ本体のソース、`ccgen` がその実行ファイルです。

この `ccgen` に、先ほどの `sample.c` と同等の処理を定義した `PEN.def` を入力します。ここではまず、`PEN.def` がどういう形をしているかだけ見ます。(理解しなくてよいです)

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

`PEN.def` を `ccgen` に与えると `PEN.c` が生成されます。これをコンパイルして `PEN.exe` を作れば、`This is a pen` に対して `I am a boy. F-1 is Formula 1` を出力するプログラムが得られます。

```bash
# コンパイル、実行手順
$ ccgen PEN.def PEN.c
$ gcc -o PEN.exe PEN.c
$ PEN.exe pen.txt penout.txt
```

<table class="plain-table">
  <thead>
    <tr>
      <th>入力ファイルの中身（pen.txt）</th>
      <th>出力ファイルの中身（penout.txt）</th>
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
      <th>入力ファイルの中身（pen.txt）</th>
      <th>出力ファイルの中身（penout.txt）</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td>It is a ball pencil</td>
      <td>It is a ball</td>
    </tr>
  </tbody>
</table>

ここまでで、コンパイラコンパイラが「定義から C ソースを作る道具」であることは見えてきたはずです。

# 定義ファイル

ここからは `PEN.def` の中身を見ます。

全体の骨格は次の形です。

```
START(PEN0) ... END
```

先頭に `START` を置き、括弧の中に言語名を書きます。ここでは `PEN0` です。末尾は `END` で閉じます。

`START(PEN0)` と `END` の間には、次の形の定義を並べます。

```
超変数 : 定義内容
;
```

コロン `:` の左を超変数、右をその定義内容とします。定義の終わりはセミコロン `;` です。右辺は `|` で区切って複数書けます。`|` は OR、つまり「または」を表します。

`PEN.def` では、`PEN0`、`THIS`、`PEN` の 3 つの超変数を定義しています。

基本形は次の通りです。

```c
超変数A : 構文規則 { 生成規則 } ;

超変数B : 構文規則1 { 生成規則1 }
        | 構文規則2 { 生成規則2 }
        | 構文規則3 { 生成規則3 }
;
```

ここでは `構文規則 { 生成規則 }` をひとまとまりの単位として扱い、これを便宜上、構文生成セットと呼びましょう。

- 構文規則: 入力文字列と照合するためのパターン
- 生成規則: その構文規則にマッチしたときに実行する処理

生成規則が行うのは、最終的には `現在の出力ファイル` (後述)への書き込みです。構文生成セットは `|` で複数並べられます。

まずは生成規則を脇に置き、構文規則だけ見ます。

<div class="codeblock-title">PEN.def（構文規則のみ）</div>

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

`START(PEN0)` と書くと、入力との照合は `PEN0` から始まります。

<div class="codeblock-title">PEN0 の定義</div>

```c
PEN0 : THIS " is a " PEN
;
```

右辺の `" is a "` は文字列、`THIS` と `PEN` は超変数です。

要するに `PEN0` は、「`THIS` で定義したパターン + 文字列 `" is a "` + `PEN` で定義したパターン」という形の入力を受理します。

次に `THIS` です。

```c
THIS : "This"
     | "It"
;
```

`|` によって、`This` または `It` のどちらかにマッチします。したがって、ここまでで `PEN0` は `This is a ...` または `It is a ...` を受け付ける形になっています。

続いて `PEN` を見ます。

```c
PEN : "pen"
    | "ba"+"l"+"l" "pencil"
;
```

ここでの比較対象は `pen`、または `ba+l+l pencil` です。

`+` は「文字列の間に空白を入れない」という指定です。たとえば `"ba"+"l"` は、`ba` と `l` の間に空白が入ると不一致になります。

`"ba" "l"` と `"ba"+"l"` の違いを示すと、`"ba" "l"` は次のような入力にマッチします。

```c
bal
ba l
ba     l
```

一方、`"ba"+"l"` がマッチするのは `bal` だけです。

そのため `"ba"+"l"+"l"` は実質的には `"ball"` と同じですが、ここでは `+` の働きを説明するためにあえて分けて書いています。

また `"ball" "pencil"` は、2 語の間に空白があってもなくても構いません。ただし、それぞれの語の途中に空白は入れられません。対して `"ball pencil"` と 1 つの文字列として書くと、ちょうど 1 個の空白が必要です。

以上から、`PEN0` は `This is a pen` や `It is a ball pencil` のような入力を処理できます。構文だけを見れば、最初の `sample.c` と同じ発想です。

## 生成規則

次は生成規則です。まず `PEN0` を見ます。

<div class="codeblock-title">PEN0 の構文規則と生成規則</div>

```c
PEN0 : THIS " is a " PEN {
    call(1);
    print("a");
    call(2);
}
;
```

`call(1);` は `THIS` に対応し、`call(2);` は `PEN` に対応します。`call(n)` は、対応する超変数（n番目に登場する超変数）の生成規則を実行する命令です。

間にある `print("a");` は、文字列 `a` を出力ファイルへ書き込みます。

したがって `PEN0` の生成規則は、次の順序で動きます。

```c
1. THIS の生成規則を実行
2. "a" を出力
3. PEN の生成規則を実行
```

`THIS` 側は次の通りです。

```c
THIS : "This" { print("I am "); }
     | "It" { print("It is "); }
;
```

入力が `This` なら `I am `、`It` なら `It is ` を出力します。

最後に `PEN` を見ます。

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

`PEN` は 2 つの構文生成セットを持っています。

入力が `It is a ball pencil` の場合、`It is a ` までは `PEN0` の前半で処理されています。残りの `ball pencil` が `PEN` の `"ba"+"l"+"l" "pencil"` にマッチし、生成規則 `print(" ball");` が実行されるので、最終出力は `It is a ball` になります。

次に入力が `This is a pen` の場合、`PEN` 側の生成規則は次です。

```c
print(" boy. F-");
genNum();
<<
    print(" is Formula ");
    genNum();
>>
```

最初の `print(" boy. F-");` で ` boy. F-` が出力されます。

`genNum();` は数字文字列を生成する命令です。同一の生成規則の中では同じ番号が使われます。したがって、この `PEN` の生成規則にある 2 つの `genNum();` は同じ値になります。

初期値は 1 なので、この規則で最初に使う `genNum();` は 1 です。別の規則で使われる場合は次の番号になります。

つまり、この部分は実質的に次のような動作です。

```c
print(" boy. F-");
print("1");
<<
    print(" is Formula ");
    print("1");
>>
```

最後の `<< ... >>` は、その中で行った出力を、最終的に出力ファイルの末尾へ追加するための仕組みです。

この例では、まず ` boy. F-1` が出力され、その後で ` is Formula 1` が末尾に付加されます。

まとめると、全体の振る舞いは次の通りです。

<table class="plain-table">
  <thead>
    <tr>
      <th>入力ファイルの中身</th>
      <th>出力ファイルの中身</th>
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

実際に処理系で確認します。

```bash
$ ./ccgen PEN.def PEN.c
```

これで `PEN.c` が生成されます。`PEN.c` は入力文字列を構文解析し、その結果を出力する C プログラムです。

`ccgen` はその `PEN.c` を生成するので、「コンパイラを作るコンパイラ」、すなわちコンパイラコンパイラと呼ばれます。

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

`PEN.c` をコンパイルして `PEN.exe` を作ります。

```bash
$ gcc -o PEN.exe PEN.c
```

`sample.exe` と同じ挙動になることを確認します。入力ファイルは `pen.txt`、出力ファイルは `penout2.txt` とします。

```bash
$ ./PEN.exe pen.txt penout2.txt
```

期待通りの出力が得られれば、`PEN.def` から生成された `PEN.c` が意図した処理をしていることになります。

ただし、本当に理解したいのは「なぜそのように動くのか」です。実行結果だけ見れば `sample.c` と `PEN.c` はほぼ同じ働きをしますが、`PEN.c` の内部はそのままでは読み取りにくい構造になっています。

コンパイラコンパイラを理解するには、この生成後の `PEN.c` が何をしているかを追う必要があります。次章では、その中身を詳しく見ていきます。
