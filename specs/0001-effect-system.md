# 0001: Effect System

Status: Draft

## Summary

このドラフトは、Emela におけるエラーと副作用を扱う Effect System の叩き台を定義
します。

Emela では、値の型とは別に、式や関数が発生させうる effect を静的に追跡します。
I/O や host call のような platform-dependent な副作用は、Effect System に吸収された
platform capability として扱います。関数名の末尾 `!` は、その関数が effect を発生
させる可能性を表明する表層構文です。

## Motivation

Native と WASM の両方へコンパイルする言語では、I/O、host call、メモリ状態、エラー
伝播を曖昧にすると、ターゲットごとの差異がプログラムの意味に漏れます。

Effect System により、次の性質を仕様として扱えるようにします。

- pure な関数と effectful な関数を区別する。
- エラーを例外として暗黙に飛ばさず、型検査で追跡する。
- WASM host binding や Native runtime capability を明示的な境界に押し込める。
- platform ごとの差異を capability として追跡し、portable なプログラムと target
  固有のプログラムを区別する。
- 後続で非同期、リソース管理、FFI を導入する余地を残す。

## Specification

### Effects

effect は、式の評価中に値の計算以外の観測可能な振る舞いが発生しうることを表します。

初期ドラフトでは、少なくとも次の effect kind を予約します。

| Effect | 意味 |
| --- | --- |
| `Error` | 失敗または早期終了を発生させうる |
| `IO` | 外部入出力または host call を発生させうる |
| `State` | mutable state の読み書きを発生させうる |
| `Platform` | target または host が提供する capability を要求する |

effect を持たない式は pure です。

`Platform` effect の詳細は [0003: Platform Capabilities](0003-platform-capabilities.md)
で定義します。

### Function Marker

関数名の末尾 `!` は、その関数が 1 つ以上の effect を発生させうることを表明します。

```emela
fn read_line!() {
  ()
}
```

`!` を持たない関数は pure とみなされます。

pure な関数は、未処理の effect を持つ式を本体に含めてはなりません。

### Effect Signature

関数型は、値の型と effect row を持つものとして扱います。

```text
(A, B) -> C !{IO, Error}
```

上の型は、`A` と `B` を受け取り、`C` を返し、`IO` または `Error` を発生させうる
関数を表します。

effect を持たない関数型は次のように書けます。

```text
(A, B) -> C
```

型注釈の具体的な構文は、このドラフトではまだ確定しません。

### Effect Propagation

式の effect は、その部分式の effect の和集合です。

effectful な関数呼び出しは、呼び出し側の関数にも effect を伝播します。

```emela
fn load_config!() {
  read_config!()
}
```

`load_config!` は `read_config!` の effect を引き継ぎます。

### Error

エラーは `Error` effect として扱います。

エラーは暗黙の例外として pure な関数境界を越えてはなりません。エラーを値として
扱うか、handler で処理するか、呼び出し元に `Error` effect として伝播させる必要が
あります。

エラー値の表現、`Result` 型、handler 構文は後続仕様で定義します。

### Handlers

effect handler は後続仕様で定義します。

このドラフトでは、未処理の effect を持つ式は、その effect を許可する関数内でのみ
使用できます。

### Main

`main` は実行環境との境界です。

`main` が `!` を持たない場合、プログラム全体は pure な計算として扱われます。

`main!` を許可するか、`main` に特別な effect signature と capability set を与えるかは
未決定です。

## Examples

pure な関数:

```emela
fn add(x, y) {
  x + y
}
```

effectful な関数:

```emela
fn print_line!(value) {
  ()
}
```

pure な関数から effectful な関数を直接呼ぶ例。このプログラムは reject されます。

```emela
fn greet() {
  write_stdout_utf8!("hello")
}
```

effect を伝播する例:

```emela
fn greet!() {
  write_stdout_utf8!("hello")
}
```

## Compilation Notes

WASM では、`IO` または `Platform` effect を持つ関数は host import への呼び出しを
含みうるため、pure なコード生成パスと分離するべきです。

Native では、`IO` や `State` を runtime capability として明示的に受け渡す実装が
考えられます。

`Error` effect は、初期実装では tagged union による明示的な戻り値として lower
できます。暗黙の例外機構には依存しません。

## Open Questions

- `fn main!() {}` を許可するか。許可する場合、実行可能プログラムの entry point は
  `main` と `main!` のどちらを要求するか。
- `!` は effect row が空でないことを必須にするか、それとも public API 上の表明に
  留めるか。
- effect row の構文をどこまで明示させるか。
- effect handler の構文を `handle` 式にするか、関数境界の属性にするか。
- `Error` を effect として扱いつつ、`Result` 型も標準に持つべきか。
- platform capability を effect row に直接入れるか、effect row とは別の capability
  row として扱うか。
