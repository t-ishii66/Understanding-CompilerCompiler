# Appendix

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

この Appendix では、本文で後回しにした `NAME` と `QSTR` の意味を整理し、その後に資料として `t0.def`、`t3.out.c`、`comcom.h` を掲載します。

まず、`NAME` と `QSTR` の定義を再掲します。

<div class="codeblock-title">t0.def(一部)</div>

```c
NAME : CHR+NAME { call(1); call(2); }
     | CHR { call(1); }
;

QSTR : "\""+QSTR1 { print("\""); call(1); }
;

QSTR1 : "\"" { print("\""); }
      | CHR+QSTR1 { call(1); call(2); }
      | MCHR+QSTR1 { call(1); call(2); }
;

CHR : ALPH { call(1); }
    | SALPH { call(1); }
    | NUM { call(1); }
;
```


意味を一言で書くと、次の通りです。

<table class="plain-table">
  <thead>
    <tr>
      <th>超変数</th>
      <th>意味</th>
      <th>たとえば PEN.def でマッチするもの</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td>NAME</td>
      <td>名前</td>
      <td>THIS</td>
    </tr>
    <tr>
      <td>QSTR</td>
      <td>ダブルクォーテーションで囲まれた文字列</td>
      <td>" is not "</td>
    </tr>
  </tbody>
</table>

`t0.def` では、この 2 つをその役割の違いに応じて使い分けています。

`NAME` は、いまの文脈では `PEN.def` に現れる超変数名です。したがって、構文規則側にだけ登場します。`NAME` 自体の生成規則だけを見れば、文字列をそのまま出力しているように見えます。

たとえば `PEN.def` に `THIS` があるとき、それは `PEN.c` 側では関数名の一部として使われます。


<table class="plain-table">
<colgroup>
<col style="width: 50%" />
<col style="width: 50%" />
</colgroup>
<tbody>
<tr>
<td>PEN.def(一部)</td>
<td>PEN.c(一部)</td>
</tr>
<tr>
<td>

```c
PEN0 : THIS " is a " PEN {
    call(1);
    print("a");
    call(2);
    }
;
```
</td>
<td>

```c
void _PEN0(int e) {
    { _THIS(E); check(" is a "); _PEN(E); mknode(_f1);}
    }
```
</td>
</tr>
</tbody>
</table>


ここで `THIS` は、そのまま裸で出力されるのではなく、上位規則の中で `_THIS(E)` という形へ加工されています。`NAME` は単独で使われるのではなく、常に上位の規則の一部として使われるので、その前後に `_` や `(E)` などが付加されます。

`QSTR` も同様に考えられますが、こちらは構文規則側にも生成規則側にも登場できます。

- 構文規則側の `QSTR` は、`check()` に渡す比較対象になる
- 生成規則側の `QSTR` は、`print()` の引数として出力される

例を見ます。


<table class="plain-table">
<colgroup>
<col style="width: 50%" />
<col style="width: 50%" />
</colgroup>
<tbody>
<tr>
<td>PEN.def(一部)</td>
<td>PEN.c(一部)</td>
</tr>
<tr>
<td>

```c
PEN0 : THIS " is a " PEN {
    call(1);
    print("a");
    call(2);
    }
;
```
</td>
<td>

```c
void _PEN0(int e) {
    { _THIS(E); check(" is a "); _PEN(E); mknode(_f1);}
    }
void _f1(int n, int a) {call(1);print("a");call(2);}
```
</td>
</tr>
</tbody>
</table>


ここで少し紛らわしいのは、`print`の考え方です。

`PEN.def` を直接実行する例では、生成規則に書いた `print("a");` は最終的に出力ファイルへ `a` を書き出します。これは、最初の `This is a pen` の例で見た通りです。

しかし `t0.def` の立場では事情が違います。`t0.def` は `PEN.def` を入力として `PEN.c` を出力する定義です。したがって、`PEN.def` の中にある

```c
print(" is not ");
```

という入力パターンは、`t0.def` によって `PEN.c` の中へ

```c
print(" is not ");
```

という C コードとして出力されます。


状況を整理すると次のようになります。

<table class="plain-table">
  <thead>
    <tr>
      <th>定義</th>
      <th>注目している規則</th>
      <th>その規則が処理する入力</th>
      <th>その段階での出力</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td>PEN.def</td>
      <td>生成規則 print("a");</td>
      <td>This</td>
      <td>a</td>
    </tr>
    <tr>
      <td>t0.def</td>
      <td>OUT</td>
      <td>print("a");</td>
      <td>print("a");</td>
    </tr>
    <tr>
      <td>t0.def</td>
      <td>QSTR</td>
      <td>"a"</td>
      <td>"a"</td>
    </tr>
  </tbody>
</table>



---

# ソースコード

以下に、資料として `t0.def`、`t3.out.c`、`comcom.h` を掲載します。

<div class="codeblock-title">t0.def</div>

```c
START(COMCOM)
COMCOM	: "START(" NAME ")" STMTS "END"	{
					print("#include \"comcom.h\"\n");
					<<
						print("void firstcall(int e) { _");
						call(1);
						print("(e); }\n");
					>>
					call(2); }
;
STMTS	: STMT STMTS			{ call(1); call(2); }
	| STMT				{ call(1); }
;
STMT	: NAME ":" RULES ";"		{ print("void _"); call(1);
					print("(int e);\n");
					<<
						print("\n void _"); call(1);
						print("(int e) {\n"); call(2);
						print("\n }\n");
					>> }
;
RULES	: RULE "|" RULES		{ print(" if (try()) {"); call(1);
					print(" ok(); }\n  else "); call(2); }
	| RULE				{ print("{"); call(1); print("}"); }
;
RULE	: TERMS "{" OUTS "}"		{ call(1); print(" mknode(_f");
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
TERMS	: TERM TERMS			{ call(1); call(2); }
	| TERM				{ call(1); }
;
TERM	: NAME				{ print(" _"); call(1);
					print("(E);"); }
	| QSTR				{ print(" check("); call(1);
					print(");"); }
	| "+"				{ print(" combine();"); }
;
OUTS	: OUT OUTS			{ call(1); call(2); }
	| OUT				{ call(1); }
;
OUT	: "call(" NUMS ");"		{ print("call("); call(1);
					print(");"); }
	| "print(" QSTR ");"		{ print("print("); call(1);
					print(");"); }
	| "genNum();"			{ print("genNum();"); }
	| "<<" OUTS ">>"		{ print("spoolOpen();\n"); call(1);
					print("spoolClose();\n"); }
;
NAME	: CHR+NAME			{ call(1); call(2); }
	| CHR				{ call(1); }
;
QSTR	: "\""+QSTR1			{ print("\""); call(1); }
;
QSTR1	: "\""				{ print("\""); }
	| CHR+QSTR1			{ call(1); call(2); }
	| MCHR+QSTR1			{ call(1); call(2); }
;

CHR	: ALPH				{ call(1); }
	| SALPH				{ call(1); }
	| NUM				{ call(1); }
;
MCHR	: "\\"+MARK			{ print("\\"); call(1); }
	| MARK				{ call(1); }
;
NUMS	: NUM+NUMS			{ call(1); call(2); }
	| NUM				{ call(1); }
;
ALPH	: "A"				{ print("A"); }
	| "B"				{ print("B"); }
	| "C"				{ print("C"); }
	| "D"				{ print("D"); }
	| "E"				{ print("E"); }
	| "F"				{ print("F"); }
	| "G"				{ print("G"); }
	| "H"				{ print("H"); }
	| "I"				{ print("I"); }
	| "J"				{ print("J"); }
	| "K"				{ print("K"); }
	| "L"				{ print("L"); }
	| "M"				{ print("M"); }
	| "N"				{ print("N"); }
	| "O"				{ print("O"); }
	| "P"				{ print("P"); }
	| "Q"				{ print("Q"); }
	| "R"				{ print("R"); }
	| "S"				{ print("S"); }
	| "T"				{ print("T"); }
	| "U"				{ print("U"); }
	| "V"				{ print("V"); }
	| "W"				{ print("W"); }
	| "X"				{ print("X"); }
	| "Y"				{ print("Y"); }
	| "Z"				{ print("Z"); }
;
SALPH	: "a"				{ print("a"); }
	| "b"				{ print("b"); }
	| "c"				{ print("c"); }
	| "d"				{ print("d"); }
	| "e"				{ print("e"); }
	| "f"				{ print("f"); }
	| "g"				{ print("g"); }
	| "h"				{ print("h"); }
	| "i"				{ print("i"); }
	| "j"				{ print("j"); }
	| "k"				{ print("k"); }
	| "l"				{ print("l"); }
	| "m"				{ print("m"); }
	| "n"				{ print("n"); }
	| "o"				{ print("o"); }
	| "p"				{ print("p"); }
	| "q"				{ print("q"); }
	| "r"				{ print("r"); }
	| "s"				{ print("s"); }
	| "t"				{ print("t"); }
	| "u"				{ print("u"); }
	| "v"				{ print("v"); }
	| "w"				{ print("w"); }
	| "x"				{ print("x"); }
	| "y"				{ print("y"); }
	| "z"				{ print("z"); }
;
NUM	: "0"				{ print("0"); }
	| "1"				{ print("1"); }
	| "2"				{ print("2"); }
	| "3"				{ print("3"); }
	| "4"				{ print("4"); }
	| "5"				{ print("5"); }
	| "6"				{ print("6"); }
	| "7"				{ print("7"); }
	| "8"				{ print("8"); }
	| "9"				{ print("9"); }
;
MARK	: " "				{ print(" "); }
	| "!"				{ print("!"); }
	| "\""				{ print("\""); }
	| "#"				{ print("#"); }
	| "$"				{ print("$"); }
	| "%"				{ print("%"); }
	| "&"				{ print("&"); }
	| "'"				{ print("'"); }
	| "("				{ print("("); }
	| ")"				{ print(")"); }
	| "*"				{ print("*"); }
	| "+"				{ print("+"); }
	| ","				{ print(","); }
	| "-"				{ print("-"); }
	| "."				{ print("."); }
	| "/"				{ print("/"); }
	| ":"				{ print(":"); }
	| ";"				{ print(";"); }
	| "<"				{ print("<"); }
	| "="				{ print("="); }
	| ">"				{ print(">"); }
	| "?"				{ print("?"); }
	| "@"				{ print("@"); }
	| "["				{ print("["); }
	| "]"				{ print("]"); }
	| "\\"				{ print("\\"); }
	| "^"				{ print("^"); }
	| "_"				{ print("_"); }
	| "|"				{ print("|"); }
	| "~"				{ print("~"); }
	| "{"				{ print("{"); }
	| "}"				{ print("}"); }
;
END
```

<div class="codeblock-title">t3.out.c</div>

```c
#include "comcom.h"
void _COMCOM(int e);
void _STMTS(int e);
void _STMT(int e);
void _RULES(int e);
void _RULE(int e);
void _TERMS(int e);
void _TERM(int e);
void _OUTS(int e);
void _OUT(int e);
void _NAME(int e);
void _QSTR(int e);
void _QSTR1(int e);
void _CHR(int e);
void _MCHR(int e);
void _NUMS(int e);
void _ALPH(int e);
void _SALPH(int e);
void _NUM(int e);
void _MARK(int e);
void firstcall(int e) { _COMCOM(e); }

 void _COMCOM(int e) {
{ check("START("); _NAME(E); check(")"); _STMTS(E); check("END"); mknode(_f1);}
 }

 void _STMTS(int e) {
 if (try()) { _STMT(E); _STMTS(E); mknode(_f2); ok(); }
  else { _STMT(E); mknode(_f3);}
 }

 void _STMT(int e) {
{ _NAME(E); check(":"); _RULES(E); check(";"); mknode(_f4);}
 }

 void _RULES(int e) {
 if (try()) { _RULE(E); check("|"); _RULES(E); mknode(_f5); ok(); }
  else { _RULE(E); mknode(_f6);}
 }

 void _RULE(int e) {
{ _TERMS(E); check("{"); _OUTS(E); check("}"); mknode(_f7);}
 }

 void _TERMS(int e) {
 if (try()) { _TERM(E); _TERMS(E); mknode(_f8); ok(); }
  else { _TERM(E); mknode(_f9);}
 }

 void _TERM(int e) {
 if (try()) { _NAME(E); mknode(_f10); ok(); }
  else  if (try()) { _QSTR(E); mknode(_f11); ok(); }
  else { check("+"); mknode(_f12);}
 }

 void _OUTS(int e) {
 if (try()) { _OUT(E); _OUTS(E); mknode(_f13); ok(); }
  else { _OUT(E); mknode(_f14);}
 }

 void _OUT(int e) {
 if (try()) { check("call("); _NUMS(E); check(");"); mknode(_f15); ok(); }
  else  if (try()) { check("print("); _QSTR(E); check(");"); mknode(_f16); ok(); }
  else  if (try()) { check("genNum();"); mknode(_f17); ok(); }
  else { check("<<"); _OUTS(E); check(">>"); mknode(_f18);}
 }

 void _NAME(int e) {
 if (try()) { _CHR(E); combine(); _NAME(E); mknode(_f19); ok(); }
  else { _CHR(E); mknode(_f20);}
 }

 void _QSTR(int e) {
{ check("\""); combine(); _QSTR1(E); mknode(_f21);}
 }

 void _QSTR1(int e) {
 if (try()) { check("\""); mknode(_f22); ok(); }
  else  if (try()) { _CHR(E); combine(); _QSTR1(E); mknode(_f23); ok(); }
  else { _MCHR(E); combine(); _QSTR1(E); mknode(_f24);}
 }

 void _CHR(int e) {
 if (try()) { _ALPH(E); mknode(_f25); ok(); }
  else  if (try()) { _SALPH(E); mknode(_f26); ok(); }
  else { _NUM(E); mknode(_f27);}
 }

 void _MCHR(int e) {
 if (try()) { check("\\"); combine(); _MARK(E); mknode(_f28); ok(); }
  else { _MARK(E); mknode(_f29);}
 }

 void _NUMS(int e) {
 if (try()) { _NUM(E); combine(); _NUMS(E); mknode(_f30); ok(); }
  else { _NUM(E); mknode(_f31);}
 }

 void _ALPH(int e) {
 if (try()) { check("A"); mknode(_f32); ok(); }
  else  if (try()) { check("B"); mknode(_f33); ok(); }
  else  if (try()) { check("C"); mknode(_f34); ok(); }
  else  if (try()) { check("D"); mknode(_f35); ok(); }
  else  if (try()) { check("E"); mknode(_f36); ok(); }
  else  if (try()) { check("F"); mknode(_f37); ok(); }
  else  if (try()) { check("G"); mknode(_f38); ok(); }
  else  if (try()) { check("H"); mknode(_f39); ok(); }
  else  if (try()) { check("I"); mknode(_f40); ok(); }
  else  if (try()) { check("J"); mknode(_f41); ok(); }
  else  if (try()) { check("K"); mknode(_f42); ok(); }
  else  if (try()) { check("L"); mknode(_f43); ok(); }
  else  if (try()) { check("M"); mknode(_f44); ok(); }
  else  if (try()) { check("N"); mknode(_f45); ok(); }
  else  if (try()) { check("O"); mknode(_f46); ok(); }
  else  if (try()) { check("P"); mknode(_f47); ok(); }
  else  if (try()) { check("Q"); mknode(_f48); ok(); }
  else  if (try()) { check("R"); mknode(_f49); ok(); }
  else  if (try()) { check("S"); mknode(_f50); ok(); }
  else  if (try()) { check("T"); mknode(_f51); ok(); }
  else  if (try()) { check("U"); mknode(_f52); ok(); }
  else  if (try()) { check("V"); mknode(_f53); ok(); }
  else  if (try()) { check("W"); mknode(_f54); ok(); }
  else  if (try()) { check("X"); mknode(_f55); ok(); }
  else  if (try()) { check("Y"); mknode(_f56); ok(); }
  else { check("Z"); mknode(_f57);}
 }

 void _SALPH(int e) {
 if (try()) { check("a"); mknode(_f58); ok(); }
  else  if (try()) { check("b"); mknode(_f59); ok(); }
  else  if (try()) { check("c"); mknode(_f60); ok(); }
  else  if (try()) { check("d"); mknode(_f61); ok(); }
  else  if (try()) { check("e"); mknode(_f62); ok(); }
  else  if (try()) { check("f"); mknode(_f63); ok(); }
  else  if (try()) { check("g"); mknode(_f64); ok(); }
  else  if (try()) { check("h"); mknode(_f65); ok(); }
  else  if (try()) { check("i"); mknode(_f66); ok(); }
  else  if (try()) { check("j"); mknode(_f67); ok(); }
  else  if (try()) { check("k"); mknode(_f68); ok(); }
  else  if (try()) { check("l"); mknode(_f69); ok(); }
  else  if (try()) { check("m"); mknode(_f70); ok(); }
  else  if (try()) { check("n"); mknode(_f71); ok(); }
  else  if (try()) { check("o"); mknode(_f72); ok(); }
  else  if (try()) { check("p"); mknode(_f73); ok(); }
  else  if (try()) { check("q"); mknode(_f74); ok(); }
  else  if (try()) { check("r"); mknode(_f75); ok(); }
  else  if (try()) { check("s"); mknode(_f76); ok(); }
  else  if (try()) { check("t"); mknode(_f77); ok(); }
  else  if (try()) { check("u"); mknode(_f78); ok(); }
  else  if (try()) { check("v"); mknode(_f79); ok(); }
  else  if (try()) { check("w"); mknode(_f80); ok(); }
  else  if (try()) { check("x"); mknode(_f81); ok(); }
  else  if (try()) { check("y"); mknode(_f82); ok(); }
  else { check("z"); mknode(_f83);}
 }

 void _NUM(int e) {
 if (try()) { check("0"); mknode(_f84); ok(); }
  else  if (try()) { check("1"); mknode(_f85); ok(); }
  else  if (try()) { check("2"); mknode(_f86); ok(); }
  else  if (try()) { check("3"); mknode(_f87); ok(); }
  else  if (try()) { check("4"); mknode(_f88); ok(); }
  else  if (try()) { check("5"); mknode(_f89); ok(); }
  else  if (try()) { check("6"); mknode(_f90); ok(); }
  else  if (try()) { check("7"); mknode(_f91); ok(); }
  else  if (try()) { check("8"); mknode(_f92); ok(); }
  else { check("9"); mknode(_f93);}
 }

 void _MARK(int e) {
 if (try()) { check(" "); mknode(_f94); ok(); }
  else  if (try()) { check("!"); mknode(_f95); ok(); }
  else  if (try()) { check("\""); mknode(_f96); ok(); }
  else  if (try()) { check("#"); mknode(_f97); ok(); }
  else  if (try()) { check("$"); mknode(_f98); ok(); }
  else  if (try()) { check("%"); mknode(_f99); ok(); }
  else  if (try()) { check("&"); mknode(_f100); ok(); }
  else  if (try()) { check("'"); mknode(_f101); ok(); }
  else  if (try()) { check("("); mknode(_f102); ok(); }
  else  if (try()) { check(")"); mknode(_f103); ok(); }
  else  if (try()) { check("*"); mknode(_f104); ok(); }
  else  if (try()) { check("+"); mknode(_f105); ok(); }
  else  if (try()) { check(","); mknode(_f106); ok(); }
  else  if (try()) { check("-"); mknode(_f107); ok(); }
  else  if (try()) { check("."); mknode(_f108); ok(); }
  else  if (try()) { check("/"); mknode(_f109); ok(); }
  else  if (try()) { check(":"); mknode(_f110); ok(); }
  else  if (try()) { check(";"); mknode(_f111); ok(); }
  else  if (try()) { check("<"); mknode(_f112); ok(); }
  else  if (try()) { check("="); mknode(_f113); ok(); }
  else  if (try()) { check(">"); mknode(_f114); ok(); }
  else  if (try()) { check("?"); mknode(_f115); ok(); }
  else  if (try()) { check("@"); mknode(_f116); ok(); }
  else  if (try()) { check("["); mknode(_f117); ok(); }
  else  if (try()) { check("]"); mknode(_f118); ok(); }
  else  if (try()) { check("\\"); mknode(_f119); ok(); }
  else  if (try()) { check("^"); mknode(_f120); ok(); }
  else  if (try()) { check("_"); mknode(_f121); ok(); }
  else  if (try()) { check("|"); mknode(_f122); ok(); }
  else  if (try()) { check("~"); mknode(_f123); ok(); }
  else  if (try()) { check("{"); mknode(_f124); ok(); }
  else { check("}"); mknode(_f125);}
 }
void _f1(int n, int a) {print("#include \"comcom.h\"\n");spoolOpen();
print("void firstcall(int e) { _");call(1);print("(e); }\n");spoolClose();
call(2);}
void _f2(int n, int a) {call(1);call(2);}
void _f3(int n, int a) {call(1);}
void _f4(int n, int a) {print("void _");call(1);print("(int e);\n");spoolOpen();
print("\n void _");call(1);print("(int e) {\n");call(2);print("\n }\n");spoolClose();
}
void _f5(int n, int a) {print(" if (try()) {");call(1);print(" ok(); }\n  else ");call(2);}
void _f6(int n, int a) {print("{");call(1);print("}");}
void _f7(int n, int a) {call(1);print(" mknode(_f");genNum();print(");");spoolOpen();
print("void _f");genNum();print("(int n, int a) {");call(2);print("}\n");spoolClose();
}
void _f8(int n, int a) {call(1);call(2);}
void _f9(int n, int a) {call(1);}
void _f10(int n, int a) {print(" _");call(1);print("(E);");}
void _f11(int n, int a) {print(" check(");call(1);print(");");}
void _f12(int n, int a) {print(" combine();");}
void _f13(int n, int a) {call(1);call(2);}
void _f14(int n, int a) {call(1);}
void _f15(int n, int a) {print("call(");call(1);print(");");}
void _f16(int n, int a) {print("print(");call(1);print(");");}
void _f17(int n, int a) {print("genNum();");}
void _f18(int n, int a) {print("spoolOpen();\n");call(1);print("spoolClose();\n");}
void _f19(int n, int a) {call(1);call(2);}
void _f20(int n, int a) {call(1);}
void _f21(int n, int a) {print("\"");call(1);}
void _f22(int n, int a) {print("\"");}
void _f23(int n, int a) {call(1);call(2);}
void _f24(int n, int a) {call(1);call(2);}
void _f25(int n, int a) {call(1);}
void _f26(int n, int a) {call(1);}
void _f27(int n, int a) {call(1);}
void _f28(int n, int a) {print("\\");call(1);}
void _f29(int n, int a) {call(1);}
void _f30(int n, int a) {call(1);call(2);}
void _f31(int n, int a) {call(1);}
void _f32(int n, int a) {print("A");}
void _f33(int n, int a) {print("B");}
void _f34(int n, int a) {print("C");}
void _f35(int n, int a) {print("D");}
void _f36(int n, int a) {print("E");}
void _f37(int n, int a) {print("F");}
void _f38(int n, int a) {print("G");}
void _f39(int n, int a) {print("H");}
void _f40(int n, int a) {print("I");}
void _f41(int n, int a) {print("J");}
void _f42(int n, int a) {print("K");}
void _f43(int n, int a) {print("L");}
void _f44(int n, int a) {print("M");}
void _f45(int n, int a) {print("N");}
void _f46(int n, int a) {print("O");}
void _f47(int n, int a) {print("P");}
void _f48(int n, int a) {print("Q");}
void _f49(int n, int a) {print("R");}
void _f50(int n, int a) {print("S");}
void _f51(int n, int a) {print("T");}
void _f52(int n, int a) {print("U");}
void _f53(int n, int a) {print("V");}
void _f54(int n, int a) {print("W");}
void _f55(int n, int a) {print("X");}
void _f56(int n, int a) {print("Y");}
void _f57(int n, int a) {print("Z");}
void _f58(int n, int a) {print("a");}
void _f59(int n, int a) {print("b");}
void _f60(int n, int a) {print("c");}
void _f61(int n, int a) {print("d");}
void _f62(int n, int a) {print("e");}
void _f63(int n, int a) {print("f");}
void _f64(int n, int a) {print("g");}
void _f65(int n, int a) {print("h");}
void _f66(int n, int a) {print("i");}
void _f67(int n, int a) {print("j");}
void _f68(int n, int a) {print("k");}
void _f69(int n, int a) {print("l");}
void _f70(int n, int a) {print("m");}
void _f71(int n, int a) {print("n");}
void _f72(int n, int a) {print("o");}
void _f73(int n, int a) {print("p");}
void _f74(int n, int a) {print("q");}
void _f75(int n, int a) {print("r");}
void _f76(int n, int a) {print("s");}
void _f77(int n, int a) {print("t");}
void _f78(int n, int a) {print("u");}
void _f79(int n, int a) {print("v");}
void _f80(int n, int a) {print("w");}
void _f81(int n, int a) {print("x");}
void _f82(int n, int a) {print("y");}
void _f83(int n, int a) {print("z");}
void _f84(int n, int a) {print("0");}
void _f85(int n, int a) {print("1");}
void _f86(int n, int a) {print("2");}
void _f87(int n, int a) {print("3");}
void _f88(int n, int a) {print("4");}
void _f89(int n, int a) {print("5");}
void _f90(int n, int a) {print("6");}
void _f91(int n, int a) {print("7");}
void _f92(int n, int a) {print("8");}
void _f93(int n, int a) {print("9");}
void _f94(int n, int a) {print(" ");}
void _f95(int n, int a) {print("!");}
void _f96(int n, int a) {print("\"");}
void _f97(int n, int a) {print("#");}
void _f98(int n, int a) {print("$");}
void _f99(int n, int a) {print("%");}
void _f100(int n, int a) {print("&");}
void _f101(int n, int a) {print("'");}
void _f102(int n, int a) {print("(");}
void _f103(int n, int a) {print(")");}
void _f104(int n, int a) {print("*");}
void _f105(int n, int a) {print("+");}
void _f106(int n, int a) {print(",");}
void _f107(int n, int a) {print("-");}
void _f108(int n, int a) {print(".");}
void _f109(int n, int a) {print("/");}
void _f110(int n, int a) {print(":");}
void _f111(int n, int a) {print(";");}
void _f112(int n, int a) {print("<");}
void _f113(int n, int a) {print("=");}
void _f114(int n, int a) {print(">");}
void _f115(int n, int a) {print("?");}
void _f116(int n, int a) {print("@");}
void _f117(int n, int a) {print("[");}
void _f118(int n, int a) {print("]");}
void _f119(int n, int a) {print("\\");}
void _f120(int n, int a) {print("^");}
void _f121(int n, int a) {print("_");}
void _f122(int n, int a) {print("|");}
void _f123(int n, int a) {print("~");}
void _f124(int n, int a) {print("{");}
void _f125(int n, int a) {print("}");}
```

<div class="codeblock-title">comcom.h</div>

```c
/*
	Original Source code is written in Interface 1995/12 CQ-publishing
	company. www.cqpub.co.jp

	This sorce code is drived version.
*/

#include <stdio.h>
#include <stdlib.h>
#include <ctype.h>
#include <setjmp.h>
#include <string.h>
#include <unistd.h>

typedef unsigned char	byte;
typedef unsigned int	uint;

#define TRUE	1
#define FALSE	0

/* Forward declarations */
void back(void);
void spoolOutput(void);
void firstcall(int e);

static FILE* FI = 0;
static FILE* FO = 0;
static FILE* FS1 = 0;
static FILE* FS2 = 0;

static int K = TRUE;
static int F = 0;
static int D = 0;
static int verbose = 0;

#define IB_SIZE	50000
static byte IB[IB_SIZE];
static int I = 0;

#define BS_SIZE	10000
struct {
	jmp_buf	j;
	int	e;
	int	t;
	int	i;
	int	k;
} BS[BS_SIZE];
static int B = 0;

#define ES_SIZE	10000
static int ES[ES_SIZE];
static int E = 0;

#define TS_SIZE	20000
union {
	int	(*f)();
	int	n;
} TS[TS_SIZE];
static int T = 0;

static long maxI = 0;
static long maxO = 0;
static long maxS = 0;
static long cntB = 0;
static long cntI = 0;
static int maxB = 0;
static int maxE = 0;
static int maxT = 0;

void report()
{
	printf("source size %8ld\n", maxI);
	printf("object size %8ld\n", maxO);
	printf("spool  size %8ld\n", maxS);
	printf("back  track %8ld\n", cntB);
	printf("source read %8ld\n", cntI);
	printf("BS max size %8d\n", maxB);
	printf("ES max size %8d\n", maxE);
	printf("TS max size %8d\n", maxT);
}

void stop(char* s)
{
	spoolOutput();
	fclose(FS1);
	fclose(FS2);
	if (verbose) {
		fclose(FO);
		fclose(FI);
	}
	unlink("SPOOL1.$$$");
	unlink("SPOOL2.$$$");
	if (verbose) {
		report();
		printf("\n%s\n", s);
	}
	exit(0);
}
void abend(char* s)
{
	printf("\n### %s error ###\n", s);
	stop("** abnormal end **");
}

void init(int argc, char* argv[])
{
	uint	c;

	if (argc == 1) {
		FI = stdin;
		FO = stdout;
	} else {
		verbose = 1;
		printf("source file = %s\n", argv[1]);
		printf("object file = %s\n", argv[2]);

		FI = fopen(argv[1], "r");
		if (!FI) abend("input file open");

		FO = fopen(argv[2], "w");
		if (!FO) abend("output file open");
	}

	FS1 = fopen("SPOOL1.$$$", "w");
	if (!FS1) abend("spool1 file open");

	FS2 = fopen("SPOOL2.$$$", "w");
	if (!FS2) abend("spool2 file open");

	if (verbose)
		printf("\n--- source list ---\n");
	while (!feof(FI)) {
		c = fgetc(FI);
		if (maxI >= IB_SIZE)
			abend("IB over");
		IB[maxI ++] = c;
	}
}

uint fileInput()
{
	static int i = 0;
	cntI ++;
	if (verbose && i == I && i < maxI)
		putchar(IB[i ++]);
	return (uint)((I < maxI) ? IB[I ++] : EOF);
}

#define combine() K=FALSE

void skipSpace(byte s)
{
	uint c;
	if (!K) {
		K = TRUE;
		return;
	}
	while ((c = fileInput()) != EOF && (byte)c != s && isspace(c))
		;
	-- I;
}

void check(const byte* s)
{
	skipSpace(*s);
	while (*s) {
		char	c = fileInput();
		if (*s ++ != c)
			back();
	}
}

#define spoolOpen()	 F++
#define spoolClose()	 F--

void print(const byte* s)
{
	int z = strlen((const char*)s);
	if (F == 0) {
		fputs((const char*)s, FO); maxO += z;
	} else if (F == 1) {
		fputs((const char*)s, FS1); maxS += z;
	} else {
		fputs((const char*)s, FS2); maxS += z;
	}
}

#define genNum()	genNumber(&a)

void genNumber(uint* d)
{
	static char s[6];
	if (*d == 0)
		*d = ++ D;
	sprintf(s, "%u", *d);
	print((byte*)s);
}

void spoolOutput()
{
	uint c;
	fclose(FS1);
	FS1 = fopen("SPOOL1.$$$", "r");
	while (!feof(FS1)) {
		c = fgetc(FS1);
		if (c == EOF)
			break;
		putc(c, FO);
	}
	fclose(FS2);
	FS2 = fopen("SPOOL2.$$$", "r");
	while (!feof(FS2)) {
		c = fgetc(FS2);
		if (c == EOF)
			break;
		putc(c, FO);
	}
	maxO += maxS;
}

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

void back()
{
	if (--B < 0)
		abend("syntax");
	E = BS[B].e;
	T = BS[B].t;
	I = BS[B].i;
	K = BS[B].k;
	++ cntB;
	longjmp(BS[B].j, 1);
}

void storeFunc(void (*f)(int, int))
{
	if (T > TS_SIZE)
		abend("TS over");
	TS[T ++].f = (int (*)())f;
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

#define call(a)		call2(n, a)

void call2(int n, int b)
{
	int m;
	if (b > TS[n + 1].n || b < 1)
		abend("branch no");

	m = (TS[n + 1 + b].n);
	((void (*)(int, int))TS[m].f)(m, 0);
}

void exec()
{
	int t;
	if (E != 1)
		abend("exec");
	t = ES[-- E];
	((void (*)(int, int))TS[t].f)(t, 0);
}

int main(int argc, char* argv[])
{
	if (argc != 1 && argc != 3) {
		printf(" Usage : %s [input output]\n", argv[0]);
		exit(1);
	}

	init(argc, argv);
	firstcall(E);
	exec();
	stop("** normal end **");
	return 0;
}
```

# 終わりに

ここまでで、自己定義されたコンパイラコンパイラの構造を一通り追いました。

普段は `yacc` や `bison` のような既製ツールを使うことが多く、その内部や、さらに自己生成という段階まで踏み込む機会はほとんどありません。この題材は実用上の近道ではありませんが、論理だけで組み上がる仕組みを最後まで追えるという点に価値があります。

この文書の役割は、その構造を最後まで見通せる形で示すことです。ここまで読み切れたなら、目的は十分に達しています。

# 参考文献

- インターフェース95年12月号特集 CQ出版
- コンパイラ構成法 原田賢一著 共立出版
- コンパイラI・II A.V.エイホ他著 サイエンス社


# クレジット

- 著者: t-ishii66
- 監修: GPT5.3 Codex
- 英語翻訳: GPT5.3 Codex, t-ishii66

Copyright(C) 2026 t-ishii66. All rights reserved.