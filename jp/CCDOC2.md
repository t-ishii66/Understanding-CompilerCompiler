# 生成ファイルの構造

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

まず、`ccgen` が出力する C ソースの先頭は、必ず次のような形になります。

<div class="codeblock-title">PEN.c（一部）</div>

```c
#include "comcom.h"
void firstcall(int e) { _PEN0(e); }
```

出力されたソースは `comcom.h` を利用して動作します。`main()` も `comcom.h` 側に定義されており、最終的には `firstcall()` を呼び出すようになっています。

ここから先を理解するには、解析木 `TS[]` とエントリスタック `ES[]` という 2 つの配列を押さえる必要があります。

# 解析木とエントリスタック

この処理系の中核になるのが、解析木 `TS[]` とエントリスタック `ES[]` です。

- 解析木 `TS[]` は、整数または関数ポインタを格納する配列
- エントリスタック `ES[]` は、整数だけを格納する配列

`TS[]` は木そのものを表し、`ES[]` は「まだ親に束ねられていない節」を指すために使われます。

## 関数へのポインタ

関数ポインタが馴染みにくければ、まずは次の例だけ見てください。

<div class="codeblock-title">関数ポインタの例</div>

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
    ret = (*f)(3, 4);  /* func を呼び出す。ret は 7 */
}
```

関数ポインタは変数の一種です。宣言は次の形になります。

```c
戻り値 (*名前)(引数);
```

たとえば、整数を 2 つ受け取り、整数を返す関数 `func` は次のように宣言します。

```c
int func(int, int);
```

これに対応する関数ポインタは次です。

```c
int (*f)(int, int);
```

ここで `f = func;` と書くと、`func` のアドレスが `f` に代入されます。この時点では関数はまだ呼ばれていません。呼び出すのは次の形です。

```c
ret = (*f)(2, 3);
```

`TS[]` には、このような関数ポインタが格納されます。

## 解析木 `TS[]`

`PEN.c` は `comcom.h` をインクルードしているので、実際の構造は `comcom.h` 側にあります。まずは `TS[]` です。

<div class="codeblock-title">comcom.h（解析木を作る構造体）</div>

```c
union {
    int (*f)();
    int n;
} TS[TS_SIZE];
```

要点は単純で、`TS[]` の各要素には「関数ポインタ」か「整数」のどちらかが入ります。

| index | 解析木TS[]の中身 | アクセス方法 |
|------|------------------|--------------|
| 0 | 関数ポインタ | TS[0].f |
| 1 | 整数 | TS[1].n |
| 2 | 整数 | TS[2].n |
| 3 | 関数ポインタ | TS[3].f |
| 4 | 整数 | TS[4].n |
| ... | ... | ... |

## エントリスタック `ES[]`

<div class="codeblock-title">comcom.h（エントリスタックを作る構造）</div>

```c
int ES[ES_SIZE];
```

こちらはただの整数配列です。

| index | エントリスタックES[]の中身 | アクセス方法 |
|------|----------------------------|--------------|
| 0 | 整数 | ES[0] |
| 1 | 整数 | ES[1] |
| 2 | 整数 | ES[2] |
| 3 | 整数 | ES[3] |
| ... | ... | ... |

解析木 `TS[]` の構築には、`makeNode()` と `mknode()` が使われます。

<div class="codeblock-title">comcom.h（解析木・エントリスタック操作関連）</div>

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

ここが最初の難所です。いきなり全部理解しようとせず、具体例で追います。

## グローバル変数 `T`, `E`

`TS[]` と `ES[]` は、先頭から順に使っていきます。`T`,`E`はそれぞれ `TS[]`,`ES[]`のインデックスです。

- `T` は `TS[]` の、次に利用する位置
- `E` は `ES[]` の、次に利用する位置

どちらも初期値は 0 です。`TS[]` を 1 要素使うたびに `T` を 1 増やし、`ES[]` を 1 要素使うたびに `E` を 1 増やします。

次のサンプルで、実際にどう解析木 `TS[]`が作られるかを見ます。

<div class="codeblock-title">解析木TS[]、エントリスタックES[]を理解するためのサンプルコード</div>

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

見た目は単純ですが、`mknode(...)` と `call(...)` が重要です。

`main()` から `A(E)` が呼ばれる時点では `E = 0` です。したがって `A` の中の `e` は 0 です。`A` はまず `B(E)` を呼びます。ここでもまだ `E = 0` なので、`B(0)` が呼ばれます。

`B` は `D(E)` を呼びます。したがって最初に追うのは `D(0)` です。

`D` の中身は `mknode(_f4);` だけです。これはマクロで、実際には次になります。

```c
#define mknode(f) { void f(int,int); makeNode(f, e); }
```

`D(0)` の中では `e = 0` なので、実際に実行されるのは次です。

```c
makeNode(_f4, 0);
```

`makeNode()` をもう一度載せます。

<div class="codeblock-title">comcom.h（関数makeNode）</div>

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

呼び出し時点の変数は次の通りです。

| makeNodeが扱う変数 | makeNode実行開始（初期値） | makeNode処理終了時（参考） |
|--------------------|----------------------------|--------------------|
| n | _f4 | _f4 |
| e | 0 | 0 |
| t | 0 | 0 |
| T | 0 | 2 |
| E | 0 | 1 |

最初に `storeFunc(_f4)` が実行されるので、`TS[]` に `_f4` が積まれ、`T` は 1 になります。

<div class="codeblock-title">comcom.h（関数storeFunc）</div>

```c
void storeFunc(int (*f)())
{
    TS[T ++].f = f;
}
```

| index | 解析木TS[] | Tの値 | Eの値 | エントリスタックES[] |
|------|------------|------|------|----------------------|
| 0 | _f4 |  | ○ |  |
| 1 |  | ○ |  |  |

次に `storeInt(E - e)` が実行されます。まだ `E = 0`, `e = 0` なので、`storeInt(0)` です。

<div class="codeblock-title">comcom.h（関数storeInt）</div>

```c
storeInt(int n) {
    TS[T++].n = n;
}
```

| index | 解析木TS[] | Tの値 | Eの値 | エントリスタックES[] |
|------|------------|------|------|----------------------|
| 0 | _f4 |  | ○ |  |
| 1 | 0 |  |  |  |
| 2 |  | ○ |  |  |

ここで積まれた `0` の意味は、あとで明らかになります。

この時点では `E = e = 0` なので `for (i = e; i < E; ++i)` は 1 回も実行されません。続く `E = e;` も変化なしです。

最後に `ES[E ++] = t;` が実行されます。このとき `t = 0` なので、`ES[0] = 0` が入ります。

| index | 解析木TS[] | Tの値 | Eの値 | エントリスタックES[] |
|------|------------|------|------|----------------------|
| 0 | _f4 |  |  | 0 |
| 1 | 0 |  | ○ |  |
| 2 |  | ○ |  |  |

これで `mknode(_f4);` が終わりました。結果として `T = 2`, `E = 1` です。

次は `B(0)` の中の `F(E)` です。この時点で `E = 1` なので、実際には `F(1)` が呼ばれます。つまり `F` の中の `e` は 1 です。

`F` の中身は `mknode(_f5);` だけなので、`makeNode(_f5, 1);` が実行されます。呼び出し時点の変数は次です。

| makeNodeが扱う変数 | makeNode実行開始（初期値） | makeNode処理終了時（参考） |
|--------------------|----------------------------|--------------------|
| n | _f5 | _f5 |
| e | 1 | 1 |
| t | 2 | 2 |
| T | 2 | 4 |
| E | 1 | 2 |


最初に `_f5` を `TS[]` に積みます。

| index | 解析木TS[] | Tの値 | Eの値 | エントリスタックES[] |
|------|------------|------|------|----------------------|
| 0 | _f4 |  |  | 0 |
| 1 | 0 |  | ○ |  |
| 2 | _f5 |  |  |  |
| 3 |  | ○ |  |  |

続いて `storeInt(E - e)` です。いまは `E = 1`, `e = 1` なので 0 が積まれます。

| index | 解析木TS[] | Tの値 | Eの値 | エントリスタックES[] |
|------|------------|------|------|----------------------|
| 0 | _f4 |  |  | 0 |
| 1 | 0 |  | ○ |  |
| 2 | _f5 |  |  |  |
| 3 | 0 |  |  |  |
| 4 |  | ○ |  |  |

ここでも `for` は実行されません。最後に `ES[E ++] = t;` が実行され、`t = 2` が `ES[]` に積まれます。

| index | 解析木TS[] | Tの値 | Eの値 | エントリスタックES[] |
|------|------------|------|------|----------------------|
| 0 | _f4 |  |  | 0 |
| 1 | 0 |  |  | 2 |
| 2 | _f5 |  | ○ |  |
| 3 | 0 |  |  |  |
| 4 |  | ○ |  |  |

この時点で `ES[]` に積まれている `0` と `2` は、`TS[]` 内の `_f4` と `_f5` の位置を指しています。




ここで `B` に戻り、最後の `mknode(_f2);` を実行します。`B` は `A` から `B(0)` として呼ばれていたので、`B` の中の `e` は 0 のままです。つまり実際には `makeNode(_f2, 0);` が実行されます。

呼び出し時点の変数は次です。


| makeNodeが扱う変数 | makeNode実行開始（初期値） | makeNode処理終了時（参考） |
|--------------------|----------------------------|--------------------|
| n | _f2 | _f2 |
| e | 0 | 0 |
| t | 4 | 4 |
| T | 4 | 8 |
| E | 2 | 1 |

最初に `_f2` を積みます。

| index | 解析木TS[] | Tの値 | Eの値 | エントリスタックES[] |
|------|------------|------|------|----------------------|
| 0 | _f4 |  |  | 0 |
| 1 | 0 |  |  | 2 |
| 2 | _f5 |  | ○ |  |
| 3 | 0 |  |  |  |
| 4 | _f2 |  |  |  |
| 5 |  | ○ |  |  |

次に `storeInt(E - e)` が実行されます。ここでは `E = 2`, `e = 0` なので、2 が積まれます。

この `E - e` は、「その関数の実行中に消費された `ES[]` の個数」を表しています。`B` の中では `D(E)` と `F(E)` により 2 個のエントリが積まれたので、ここは 2 になります。

その後の `for (i = e; i < E; ++i)` は、`ES[]` に積まれている値を `TS[]` 側へコピーします。今回は `0` と `2` がコピーされます。

| index | 解析木TS[] | Tの値 | Eの値 | エントリスタックES[] |
|------|------------|------|------|----------------------|
| 0 | _f4 |  |  | 0 (_f4を指す) |
| 1 | 0 |  |  | 2 (_f5を指す) |
| 2 | _f5 |  | ○ |  |
| 3 | 0 |  |  |  |
| 4 | _f2 |  |  |  |
| 5 | 2 |  |  |  |
| 6 | 0 (_f4を指す) |  |  |  |
| 7 | 2 (_f5を指す) |  |  |  |
| 8 |  | ○ |  |  |

ここで分かることは、`TS[]` の 1 つの節が次の形を取るということです。

```c
[関数, 枝の本数, 枝1, 枝2, ...]
```

`_f2` の場合は、`_f4` と `_f5` を枝に持つ節になっています。

続く `E = e;` は、`B` の実行中に消費していた `ES[]` を巻き戻す処理です。ここでは `E = 0` になり、`B` が使っていた 2 個のエントリは解放されます。

最後に `ES[E ++] = t;` が実行されます。ここで `t = 4` は、最初に `TS[]` に積んだ `_f2` の位置です。つまり `ES[]` には「`_f2` を指す値」が 1 個だけ残ります。

| index | 解析木TS[] | Tの値 | Eの値 | エントリスタックES[] |
|------|------------|------|------|----------------------|
| 0 | _f4 |  |  | 4 (_f2を指す) |
| 1 | 0 |  | ○ |  |
| 2 | _f5 |  |  |  |
| 3 | 0 |  |  |  |
| 4 | _f2 |  |  |  |
| 5 | 2 |  |  |  |
| 6 | 0 (_f4を指す) |  |  |  |
| 7 | 2 (_f5を指す) |  |  |  |
| 8 |  | ○ |  |  |

ここまで来ると構造が見えてきます。

- `TS[]` には節が積まれていく
- `ES[]` には、まだ親に束ねられていない節だけが残る
- 1 つの関数の処理が終わると、結果として `ES[]` の消費は必ず 1 個にまとまる






今度は `A` に戻って `C(E)` を実行します。この時点で `E = 1` なので、`C(1)` が呼ばれます。`C` の中身は `mknode(_f3);` だけです。

呼び出し時点の変数は次の通りです。

| makeNodeが扱う変数 | makeNode実行開始（初期値） | makeNode処理終了時（参考） |
|--------------------|----------------------------|--------------------|
| n | _f3 | _f3 |
| e | 1 | 1 |
| t | 8 | 8 |
| T | 8 | 10 |
| E | 1 | 2 |

`C` は内部で `ES[]` を消費しないので、`storeInt(E - e)` では 0 が積まれ、最後に `ES[]` には `_f3` を指す値が 1 個追加されます。

結果は次です。

| index | 解析木TS[] | Tの値 | Eの値 | エントリスタックES[] |
|------|------------|------|------|----------------------|
| 0 | _f4 |  |  | 4 (_f2を指す) |
| 1 | 0 |  |  | 8 (_f3を指す) |
| 2 | _f5 |  | ○ |  |
| 3 | 0 |  |  |  |
| 4 | _f2 |  |  |  |
| 5 | 2 |  |  |  |
| 6 | 0 (_f4を指す) |  |  |  |
| 7 | 2 (_f5を指す) |  |  |  |
| 8 | _f3 |  |  |  |
| 9 | 0 |  |  |  |
| 10 |  | ○ |  |  |

最後に `A` の `mknode(_f1);` を実行します。`A` は `main()` から `A(0)` として呼ばれていたので、`e = 0` です。したがって `makeNode(_f1, 0);` が実行されます。

呼び出し時点の変数は次です。

| makeNodeが扱う変数 | makeNode実行開始（初期値） | makeNode処理終了時（参考） |
|--------------------|----------------------------|--------------------|
| n | _f1 | _f1 |
| e | 0 | 0 |
| t | 10 | 10 |
| T | 10 | 14 |
| E | 2 | 1 |

ここでも `E - e = 2` なので、`A` の中で生まれた 2 個の節、すなわち `_f2` と `_f3` が `_f1` の枝として束ねられます。

最終的な状態は次の通りです。

| index | 解析木TS[] | Tの値 | Eの値 | エントリスタックES[] |
|------|------------|------|------|----------------------|
| 0 | _f4 |  |  | 10 (_f1を指す) |
| 1 | 0 |  | ○ |  |
| 2 | _f5 |  |  |  |
| 3 | 0 |  |  |  |
| 4 | _f2 |  |  |  |
| 5 | 2 |  |  |  |
| 6 | 0 (_f4を指す) |  |  |  |
| 7 | 2 (_f5を指す) |  |  |  |
| 8 | _f3 |  |  |  |
| 9 | 0 |  |  |  |
| 10 | _f1 |  |  |  |
| 11 | 2 |  |  |  |
| 12 | 4 (_f2を指す) |  |  |  |
| 13 | 8 (_f3を指す) |  |  |  |
| 14 |  | ○ |  |  |

これで解析木 `TS[]` は完成です。`main()` から呼ばれた `firstcall(E)` の仕事は、この木を構築することでした。

完成した時点では、`ES[]` には 1 個だけ値が残ります。これは解析木 `TS[]` のルートを指しています。もしこの形にならなければ、構文エラーです。

## 解析木 `TS[]` の実行

次は、作った解析木 `TS[]` を実行します。`main()` は `firstcall(E)` の次に `exec()` を呼びます。

<div class="codeblock-title">comcom.h（exec の簡略版）</div>

```c
exec() {
    int t;
    t = ES[-- E];
    (*TS[t].f)(t, 0);
}
```

実際のコードでは `E == 1` を確認しています。ここでは簡略版を考えます。

いま `ES[]` に残っているのは 10 なので、`t = 10` です。したがって次が実行されます。

```c
(*TS[10].f)(10, 0);
```

`TS[10].f` は `_f1` なので、結局 `_f1(10, 0)` を呼び出していることになります。

<div class="codeblock-title">関数 _f1 のソースコード</div>

```c
_f1(int n, int d) {
    call(1); call(2);
}
```

`call` はマクロで、実体は次です。

<div class="codeblock-title">comcom.h（マクロcall）</div>

```c
#define call(a) call2(n, a)
```

したがって `_f1(10, 0)` は、実質的に次になります。

```c
call2(10, 1); call2(10, 2);
```

ここで使われるのが `call2()` です。

<div class="codeblock-title">comcom.h（関数call2）</div>

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

ここでは `TS[n + 1 + b].n` の計算が重要です。これを次のようなマクロだと思うと分かりやすくなります。

```c
#define nb(n, b) (TS[n + 1 + b].n)
```

`n` は現在の節の先頭、つまり関数ポインタの位置です。`b` はその節の何番目の枝かを表します。

たとえば `call2(10, 1)` では、`nb(10, 1)` により `TS[12].n` が読まれます。ここには 4 が入っているので、`m = 4` です。したがって最後の呼び出しは次になります。

```c
(*TS[4].f)(4, 0);
```

`TS[4].f` は `_f2` なので、結局 `_f2(4, 0)` が呼ばれます。

<div class="codeblock-title">関数 _f2 のソースコード</div>

```c
_f2(int n, int d) {
    call(1);
    call(2);
}
```

ここでも同じことが起きます。

- `call2(4, 1)` は `nb(4, 1)` により 0 を得るので、`_f4(0, 0)` を呼ぶ
- `call2(4, 2)` は `nb(4, 2)` により 2 を得るので、`_f5(2, 0)` を呼ぶ

`_f4(0, 0)` は `"Hello"` を出力し、`_f5(2, 0)` は `", "` を出力します。

続いて、元の `_f1(10, 0)` に戻り、残っていた `call2(10, 2)` が実行されます。こちらは `nb(10, 2)` により 8 を得るので、`_f3(8, 0)` を呼びます。`_f3(8, 0)` は `"World\n"` を出力します。

したがって、最終出力は次になります。

```c
Hello, World
```

## マクロ的な理解

ここまでで、解析木 `TS[]` とエントリスタック `ES[]` の細かい動き、さらに `call2(n, b)` の挙動まで追いました。

ただ、毎回ここまで細かく追っていたら大変です。一度仕組みを理解したら、次からはもっと大きな視点で見れば十分です。

<div class="codeblock-title">マクロ的な理解</div>

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

このコードを大づかみに見ると、まず `B(E)` の中で `_f4`（Dによる） と `_f5`（Fによる） が作られ、それを `_f2`(Bの末尾) が束ねます。

```c
_f2 --+--> _f4
      |
      +--> _f5
```

次に `A(E)` に戻り、`_f2`(Bによる) と `_f3`(Cによる) を `_f1` (Aの末尾)が束ねます。

```c
_f1 --+--> _f2 --+--> _f4
      |          |
      |          +--> _f5
      +--> _f3
```

ここまでが解析木 `TS[]` を作る段階です。この時点で `ES[]` には、ルート `_f1` を指す値だけが残ります。

次に実行段階に入ります。実行はルート `_f1` から始まります。

- `_f1` の `call(1)` で `_f2` が呼ばれる
- `_f2` の `call(1)`, `call(2)` により `_f4`, `_f5` が呼ばれる
- その結果、`Hello, ` が出力される
- `_f1` の `call(2)` で `_f3` が呼ばれる
- その結果、`World\n` が出力される

最終的に `Hello, World\n` が出力されます。

細部の動きを一度理解したら、以後はこのマクロな理解で十分です。重要なのは、`mknode(...)` が木を組み立て、`call(...)` がその枝をたどって実行する、という全体像です。
