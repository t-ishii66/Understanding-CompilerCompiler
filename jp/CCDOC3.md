# comcom.h の理解

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

ここからは `comcom.h` の実行時機構を見ていきます。主なテーマは 4 つです。

- `setjmp` / `longjmp` による分岐の巻き戻し
- `combine()` による空白スキップの制御
- `back()` によるバックトラック
- `print()`, `genNum()`, `spoolOpen()`, `spoolClose()` の役割

## `longjmp` の世界

C であまり使わない機能に `setjmp()` と `longjmp()` があります。動作自体は難しくありません。まずは例を見ます。

<div class="codeblock-title">longjmp のサンプルコード</div>

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

<div class="codeblock-title">実行結果</div>

```
before long jump
else statement
```

`longjmp()` が呼ばれると、対応する `setjmp()` の位置まで戻ります。

- 最初に `setjmp(a)` を呼んだときの戻り値は 0
- `longjmp(a, 1)` で戻ったときの `setjmp(a)` の戻り値は 1

この動きは少し奇妙ですが、要するに `setjmp()` が「いまの実行状態」を保存し、`longjmp()` がそこへ戻していると考えれば十分です。

ただし、すべてが自動で元に戻るわけではありません。グローバル変数などは別途復元する必要があります。ここが `comcom.h` で重要になります。

まず、`try()` と `ok()` の定義を見ます。

<div class="codeblock-title">try() / ok() の定義</div>

```c
#define try() save(),!setjmp(BS[B++].j)
#define ok() B--
```

ここで新たに出てくるのが、バックトラッキングスタック `BS[]` です。

## バックトラッキングスタック

バックトラッキングスタック `BS[]` は、`longjmp()` で戻るための情報を保存する配列です。

<div class="codeblock-title">バックトラッキングスタックの定義</div>

```c
struct {
    jmp_buf j;
    int e;
    int t;
    int i;
    int k;
} BS[BS_SIZE];
```

- `j` には `setjmp()` 用の実行環境を保存する
- `e`, `t`, `i`, `k` には、`jmp_buf` だけでは復元できない状態を保存する

ここで `e`, `t` はそれぞれグローバル変数 `E`, `T` を保存します。`E` と `T` はすでに見た、`ES[]` と `TS[]` の使用位置です。

残る `i`, `k` について見ます。

## グローバル変数 `I`, `K`

`i` に保存されるのはグローバル変数 `I` です。`I` は入力ファイルの現在位置を表します。

入力ファイルは、`init()` の中でまず `IB[]` にまとめて読み込まれ、その後は `fileInput()` が 1 文字ずつ取り出します。

<div class="codeblock-title">comcom.h（fileInput 関数の簡略版）</div>

```c
uint fileInput()
{
    return (uint)((I < max_I) ? IB[I ++] : EOF);
}
```

`I` は、次に読むべき位置です。

もう 1 つの `K` は、次のマクロで使われます。

<div class="codeblock-title">comcom.h（マクロ combine）</div>

```c
#define combine() K=FALSE
```

この `K` は、空白スキップを一時的に禁止するためのフラグです。

## 空白文字のスキップ禁止

この処理系では、入力中の空白文字は基本的に読み飛ばします。つまり、次の 2 つは同じ入力として扱います。

```text
This is a pen
```

```text
This   is a   pen
```

デフォルトでは `K = TRUE` です。この状態で `skipSpace()` を見ます。

<div class="codeblock-title">comcom.h（skipSpace 関数）</div>

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

`K = TRUE` のときは、比較対象の文字 `s` が現れる直前まで空白を読み飛ばします。

`skipSpace`を呼び出しているのが `check()` です。

<div class="codeblock-title">comcom.h（check 関数）</div>

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

たとえば `check("This")` なら、最初に `skipSpace('T')` が呼ばれます。つまり、先頭の `T` に到達するまで空白を読み飛ばし、その後で `This` を順に比較します。

ここで問題になるのは、`check(" is a ")` のように、比較対象の先頭が空白の場合です。このとき `skipSpace(' ')` が呼ばれますが、比較対象そのものが空白なので、その空白は読み飛ばされません。

これで、空白を含む文字列も正しく扱えます。

次に `K` の役割です。`combine()` を呼ぶと `K = FALSE` になります。この状態で次の `check()` が走ると、`skipSpace()` は空白を読み飛ばさず、そのまま戻ります。ただしこれは 1 回限りで、戻ると `K` はすぐ `TRUE` に戻ります。

この仕組みは、文字列を 1 文字ずつ比較するときに効いてきます。

たとえば、後半で扱う自己定義では `check("This")` の形ではなく、どうしても次のような形になる場面があります。

```c
check("T"); check("h"); check("i"); check("s");
```

このままだと、各 `check()` のたびに空白が飛ばされるので、次の 2 つがどちらも一致してしまいます。

```text
This
```

```text
T h  i s
```

これを避けるために、各文字の間へ `combine()` を挟みます。

```c
check("T"); combine(); check("h"); combine(); check("i"); combine(); check("s");
```

こうすると、`T`, `h`, `i`, `s` の間では空白が許されません。したがって、意味としては `check("This")` と同じになります。

この仕組みを踏まえて、1 文字比較の例を見ます。

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

これは、入力が `ABC` でも `A B C` でも `abc` を出力します。

次に `+` を入れます。

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

こちらは、入力が `ABC` のときだけ `abc` を出力します。`A B C` ならエラーです。

この 2 つを `ccgen` が出力した C ソースで比べると、違いは `combine();` が入るかどうかだけです。

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

つまり、構文規則で `+` を使うと、生成される C コードでは `combine();` が挟み込まれます。これが「空白を飛ばさずに連続比較する」という意味になります。

## バックトラック

次はバックトラックです。`check()` が不一致を見つけると `back()` を呼びます。これは、直前の分岐点まで処理を巻き戻して別の候補を試すための仕組みです。

まずは例を見ます。

<div class="codeblock-title">バックトラックを理解するためのサンプルコード</div>

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

`try()` と `ok()`、それに `save()` をもう一度見ます。

<div class="codeblock-title">try() / ok() の定義と save() の簡略版</div>

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

`save()` は単純です。いまの `E`, `T`, `I`, `K` を `BS[]` に退避しています。

`try()` は、その保存に続いて `setjmp()` を行います。最初の `setjmp()` は 0 を返すので、`!setjmp(...)` は真になり、`if (try()) { ... }` の本体へ入ります。

入力が `a` なら、最初の `check("a")` が成功し、`mknode(_f1)` が実行され、最後に `ok()` で `B--` が行われます。これで、その `try()` のために確保したバックトラックスタックが解放されます。

では入力が `b` の場合を考えます。

最初の `check("a")` は失敗するので、`back()` が呼ばれます。

<div class="codeblock-title">comcom.h（back 関数）</div>

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

`back()` は、直前の `try()` で保存しておいた `E`, `T`, `I`, `K` を復元し、その後 `longjmp()` で `setjmp()` の場所へ戻ります。

戻ったあとの `setjmp()` の戻り値は 1 なので、`!setjmp(...)` は偽になります。その結果、最初の `if (try())` は失敗し、制御は次の `else if (try())` に移ります。

そこで `check("b")` が実行され、今度は成功するので `_f2` が解析木 `TS[]` に積まれます。

このように、バックトラックは「失敗した地点から直前の分岐点まで状態を巻き戻し、別の候補を試す」ための仕組みです。`try()`, `back()`, `ok()` はこの 1 セットで働いています。

## その他の関数・マクロ群 `print`, `genNum`, `spoolOpen`, `spoolClose`

残っている関数やマクロは多くありません。

`print(...)` は、現在の出力ファイルへ文字列を書き出します。ここでいう「現在の出力ファイル」は、`ccgen` 実行時に指定した本来の出力ファイルか、一時ファイルのどちらかです。

ここでは便宜上、次のように呼ぶことにします。

- 本来の出力ファイル: `OUTFILE`(ccgenの第二引数のこと)
- 一時ファイル: `TMPFILE`

デフォルトでは出力先は `OUTFILE` です。`spoolOpen()` を呼ぶと出力先が `TMPFILE` に切り替わり、`spoolClose()` で再び `OUTFILE` に戻ります。

処理が全部終わると、`TMPFILE` の内容は `OUTFILE` の末尾へ付け加えられ、その後 `TMPFILE` 自体は削除されます。

<div class="codeblock-title">サンプルコード</div>

```c
print("This is ");
spoolOpen();
print("---");
print("END!\n");
spoolClose();
print("a pen.\n");
```

<div class="codeblock-title">実行結果（OUTFILE の内容）</div>

```
This is a pen.
---END!
```

次は `genNum()` です。

<div class="codeblock-title">comcom.h（genNum() マクロ、genNumber() 関数）</div>

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

`genNum()` は、実際には `genNumber(&d)` に展開されます。`d` が 0 なら新しい番号を `D` から受け取り、その後は同じ `d` を使い回します。

たとえば次のコードを考えます。

<div class="codeblock-title">サンプルコード</div>

```c
main(){ D=3; sample(0); }
sample(int d) { genNumber(&d); print("=");genNumber(&d); }
```

出力は次です。

```text
4=4
```

1 回目の `genNumber(&d)` で `d` が 4 になり、2 回目はその 4 が再利用されるからです。

ここで重要なのは、定義ファイルに書く `genNum()` と、生成された C コードで動く `genNum()` は段階が違うということです。

定義ファイルの生成規則に書かれる `genNum()` は、その場で実行される関数ではありません。あくまで `ccgen` が認識するパターンです。`ccgen` はこれを見つけると、生成する C ソースの中へ `genNum();` という文字列をそのまま書き込みます。

例を見ます。

<div class="codeblock-title">SAMPLE1.def</div>

```c
START(SAMPLE1)
SAMPLE1 : "test" { genNum(); print("="); genNum(); }
;
END
```

ここで重要なのは、以前の例にあった`+` も、今の例の `genNum()` も、その場で実行される命令ではなく、`ccgen` が出力する C ソースへ書き出すための記述だという点です。違いは出現位置だけで、`+` は構文規則側にあり、ccgenは `combine()` として出力。`genNum()` は生成規則側にあり、ccgenは `genNum();` として出力します。

`ccgen` が出力する C ソースは次のようになります。

<div class="codeblock-title">ccgen によって出力された SAMPLE1.c</div>

```c
#include "comcom.h"
void firstcall(int e) { _SAMPLE1(e); }
void _SAMPLE1(int e) {
    { check("test"); mknode(_f1);}
}
void _f1(int n, int d) {genNum();print("=");genNum();}
```

`_f1(n, d)` は必ず `call2(n, b)` を経由して呼ばれ、そのときの `d` は常に 0 です。したがって、この例では 1 回目の `genNum()` が新しい番号を取り、2 回目は同じ番号を使います。

初期状態で `D = 0` なので、入力が `test` なら出力は `1=1` です。

要するに、1 つの生成規則の中にある `genNum();` どうしは必ず同じ番号になりますが、別の生成規則の `genNum();` と同じ番号になることはありません。

`print()` も同じ考え方です。定義ファイルの生成規則に書かれた `print(...)` は、その場で意味を持つ関数呼び出しではなく、`ccgen` が生成する C ソースへそのまま書き出す対象です。実際の実行は、生成後の C プログラムが `comcom.h` を使って行います。
