# 0008: Imports

Status: Draft

## Summary

このドラフトは、Emela のトップレベル `import` 宣言を定義します。

`import` は、現在のプログラム外で定義された名前をトップレベルスコープへ導入する
ための構文です。導入された名前は、通常のトップレベル関数と同じ名前解決規則に参加
します。

この仕様は、標準ライブラリの具体的な内容、package 配布形式、runtime 実装を定義
しません。標準ライブラリが存在する場合も、この仕様で定義する import 機構を使って
参照できる外部 package の一種として扱えます。

## Motivation

Emela は、小さな core language と外部で提供される機能を分けて発展させます。

platform capability、host import、標準的な runtime helper、将来の package/module
system を扱うには、プログラム外の宣言を明示的に参照する構文が必要です。

`import` を導入することで、次の性質を仕様として扱えます。

- 外部の名前を使っていることがソース上で明示される。
- 外部関数の型、effect、capability requirement を通常の呼び出しと同じ規則で検査
  できる。
- package/module 解決と target/runtime の実装選択を、関数呼び出しの意味から分離
  できる。

## Specification

### Import Declarations

`import` 宣言はトップレベル item です。

```text
program =
  top_level_item*

top_level_item =
  import_decl
  | function
  | struct_decl
  | enum_decl

import_decl =
  import import_path

import_path =
  identifier (. identifier)* !?
```

`import_path` は 2 要素以上を MUST 持ちます。最初の要素は package name です。
最後の要素は現在のトップレベルスコープへ導入される item name です。

```emela
import platform.io.print_i32!
```

上の宣言は、package `platform` の module path `io` にある `print_i32!` を現在の
トップレベルスコープへ `print_i32!` という名前で導入します。

`!` は import path の最後の item name にのみ付けられます。これは関数名の effect
marker と同じ表層構文です。

### Name Resolution

関数呼び出し `name(args...)` は、次のいずれか 1 つへ解決されなければなりません。

- 同じプログラム内のトップレベル関数。
- `import` 宣言で導入された外部関数。

どちらにも解決できない呼び出しは MUST reject されます。複数の候補に解決できる
曖昧な名前も MUST reject されます。

同じトップレベル名を複数回導入してはなりません。import された名前と同じ名前の
トップレベル関数を同じプログラムで定義することも MUST reject されます。

このドラフトでは alias import、glob import、relative import、re-export は定義
しません。

### External Declarations

import された関数は、現在のソースファイル内に関数本体を持ちません。

コンパイラは、import path に対応する外部 declaration を解決しなければなりません。
外部 declaration は少なくとも次の情報を提供します。

```text
ExternalFunction =
  package path
  item name
  parameter types
  return type
  effect information
  capability requirement, if any
```

外部 declaration が見つからない import は MUST reject されます。

import された関数の呼び出しは、外部 declaration の型、effect、capability requirement
を使って検査します。effect と capability の伝播規則は、通常のトップレベル関数呼び
出しと同じです。

### Package Identity

このドラフトでは、package name は import path の先頭識別子です。

package name の探索場所、version 解決、package manifest の形式、同一 package 内の
module layout はこの仕様では定義しません。これらは build system または後続仕様が
定義します。

### Runtime Implementation

import は名前解決と型検査のための構文です。import された外部関数をどのように code
generation するかは、外部 declaration と target/runtime によって決まります。

target/runtime は、外部関数ごとに次のいずれかを提供できます。

- compiler backend による direct lowering。
- runtime symbol への external call。
- host import への lowering。
- 未実装であることを示す compile-time error。

capability check と runtime 実装の有無は別の判定です。target が必要な capability を
提供していても、該当する外部関数の lowering が未実装であれば、code generation は
失敗してもよいです。

### Initial Platform Imports

初期実装は、compiler-known な `platform` package を持ってよいです。この package は
外部 declaration registry として扱われ、通常の package 配布形式や標準ライブラリの
安定 ABI を意味しません。

初期の `platform` package は、少なくとも次の外部関数を宣言できます。

| Import path | Type | Capability |
| --- | --- | --- |
| `platform.io.print_i32!` | `(I32) -> Unit` | `Stdout` |
| `platform.io.print_bool!` | `(Bool) -> Unit` | `Stdout` |
| `platform.clock.now_i32!` | `() -> I32` | `Clock` |

これらの関数は effectful です。関数名に `!` を持ち、上表の capability requirement を
通常の関数呼び出しと同じ規則で呼び出し元へ伝播します。

`platform` package の具体的な runtime symbol、host import name、ABI、時刻の単位は
この仕様では固定しません。

## Examples

外部関数を import して呼び出す例:

```emela
import platform.io.print_i32!

fn main!() {
  print_i32!(42)
}
```

portable な関数だけを持つプログラムでは import は不要です。

```emela
fn main() -> I32 {
  40 + 2
}
```

## Compilation Notes

実装は、型検査前に import 宣言を外部 function signature としてトップレベル関数環境へ
登録できます。

外部 declaration の保存形式は実装依存です。初期実装では compiler 内部の registry を
使ってもよく、将来は package manifest や compiled metadata に置き換えられます。

## Open Questions

- alias import の構文を導入するか。
- import path の区切りに `.` を使い続けるか。
- package manifest と compiled metadata の形式をどう定義するか。
- 型、struct、enum、trait の import も同じ構文で扱うか。
- import された外部関数の ABI と symbol name をどの仕様で固定するか。
