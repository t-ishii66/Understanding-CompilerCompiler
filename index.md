---
title: "Understanding CompilerCompiler - コンパイラコンパイラの仕組みを学ぶ"
description: "コンパイラコンパイラ（compiler-compiler）の仕組みを、yacc/bison風の小さな処理系を題材に、構文解析・生成規則・バックトラック・自己定義まで順を追って解説するドキュメントサイトです。"
keywords:
  - コンパイラコンパイラ
  - compiler-compiler
  - 構文解析
  - parser generator
  - yacc
  - bison
  - 生成規則
  - バックトラック
  - コード生成
  - 自己定義
  - メタコンパイラ
  - metacompiler
  - BNF
  - 形式文法
  - formal grammar
---

![](top.png)

# Understanding CompilerCompiler

コンパイラコンパイラ（compiler-compiler）の仕組みを学ぶドキュメントサイト

コンパイラコンパイラとは、言語の文法や生成規則を記述すると、その定義に従って動くコンパイラや変換器を作るための処理系です。代表的な例としては yacc や bison があります。このドキュメントは、C言語の基本的な文法とプログラムの流れを理解しているプログラマーを主な読者対象としています。そのうえで、小さなコンパイラコンパイラを題材に、文法定義からコード生成がどのように成り立つのかを、具体例を通して順を追って理解できます。構文解析、生成規則、バックトラック、出力制御、そして処理系が自分自身を記述できる自己定義まで、仕組みを内側から見通せるように構成されています。

A compiler-compiler is a system that takes a description of a language, such as its grammar and generation rules, and produces a compiler or translator that follows that definition. Well-known examples include yacc and bison. This documentation is primarily intended for programmers who are already comfortable with basic C syntax and program flow. From there, it uses a small compiler-compiler to show, step by step, how grammar-based code generation actually works. Through concrete examples, it explains parsing, generation rules, backtracking, output control, and finally self-definition, so the reader can see the whole mechanism from the inside.

- [Japanese / 日本語](./jp/)
- [English](./en/)

## Structure

- `jp/`: Japanese documents
- `en/`: English documents

## Notes

- This site is published with GitHub Pages.
- Source files are managed in this Git repository.


## Credits

- Author: t-ishii66
- Supervisor: GPT5.3 Codex
- English translation: GPT5.3 Codex, t-ishii66

Copyright(C) 2026 t-ishii66. All rights reserved.
