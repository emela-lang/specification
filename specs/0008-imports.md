# 0008: Imports

Status: Draft

## Summary

このドラフトは、Emela のトップレベル `import` 宣言を定義します。

`import` は、現在のプログラム外で定義された名前をトップレベルスコープへ導入する
ための構文です。導入された名前は、通常のトップレベル関数と同じ名前解決規則に参加
します。

この仕様は、標準ライブラリの具体的な内容や runtime 実装を定義しません。
ただし、source package を import 解決へ参加させるための最小の package manifest
形式を定義します。標準ライブラリが存在する場合も、この仕様で定義する import 機構を
使って参照できる外部 package の一種として扱えます。

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
import std.io.write_stdout_utf8!
```

上の宣言は、package `std` の module path `io` にある `write_stdout_utf8!` を現在の
トップレベルスコープへ `write_stdout_utf8!` という名前で導入します。

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

### Standard Library Source Imports

実装は、package name が `std` の import を標準ライブラリ source package として
解決してよいです。

```emela
import std.io.write_stdout_utf8!
```

この import は、標準ライブラリ root の `std/io.emel` にある `write_stdout_utf8!` と、その
実装が呼び出す依存だけを現在の compilation unit へ導入します。`std.<module>.<item>`
は対応する `std/<module>.emel` source file を読み込みますが、同じ module 内の未使用
export や未使用 `platform.*` import は要求しません。したがって、組み込み環境や
`wasm32-unknown-unknown` のような backend は、ユーザーが import していない stdlib API
まで実装する必要はありません。

`std.*` module 内の `platform.*` import は backend/platform implementation API への
依存として扱います。ユーザーコードは public API として `std.*` を import します。
通常の user source が `platform.*` を直接 import することは MUST reject されます。
Native libc 呼び出しや JavaScript binding などの実装差分は backend descriptor の
`platform.*` binding で吸収します。

標準ライブラリ module が存在しない場合、または指定された item を export しない場合、
compiler は import を MUST reject します。

### Source Packages

compiler は、外部 source package directory を import 解決へ追加できてよいです。
source package directory は root に `emela-package.json` を持ちます。

```json
{
  "name": "math",
  "source": "src"
}
```

- `name` は import path の package name と一致します。
- `source` は package root から見た source root directory です。

たとえば package `math` が `source: "src"` を宣言する場合、次の import:

```emela
import math.ops.add_one
```

は package root の `src/ops.emel` を読み、module 内の top-level item `add_one` を
現在の compilation unit へ導入します。

package manager は、この source package directory を取得、展開、固定する責務を持ちます。
compiler は package の取得や version 解決を行わず、与えられた package directory を
静的に読み込むだけでよいです。

同じ package name を複数回解決へ追加してはなりません。duplicate package は MUST reject
されます。

package `std` が source package として与えられた場合、compiler は組み込み stdlib より
その package を優先してよいです。package `std` 内の `platform.*` import は stdlib
internal import として扱われます。

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

### Backend Descriptor Declarations

実装は、外部 declaration を backend descriptor から読み込めます。descriptor は
profile 名、backend kind、ABI version、起動 command または built-in backend kind、
runtime/target、capability provider、backend binding をまとめて宣言します。たとえば
`js-node`、`js-bun`、`native-aarch64-apple-darwin` はそれぞれ別 profile として表せます。

```json
{
  "name": "js-node",
  "backend": "js",
  "abi_version": 1,
  "command": ["emela-node-backend"],
  "runtime": "node",
  "capabilities": ["Stdout", "Stdin"],
  "externs": [
    {
      "path": ["platform", "io"],
      "name": "_write_stdout_utf8!",
      "params": ["String"],
      "return": "Result<Unit, PlatformError>",
      "effectful": true,
      "capabilities": ["Stdout"],
      "bindings": {
        "js": {
          "symbol": "__emela_write_stdout_utf8"
        }
      }
    }
  ]
}
```

`path` は import path の package/module 部分です。`name` は最後の item name で、
effectful な外部関数は通常の関数名と同じく `!` を含めます。

`params` と `return` は外部関数の型です。初期 descriptor 形式では primitive type
`I32`, `Bool`, `String`, `Unit`、named type、`Result<T, E>` 形式の generic type
application を MUST support します。function type を descriptor にどう表すかは後続仕様で
定義します。

`effectful` は呼び出しが effectful かどうかを表します。`capabilities` はその外部
関数を呼ぶために必要な capability set です。descriptor 内で未知の type、未知の
capability、重複する `(path, name)` を見つけた場合、compiler は MUST reject します。

### Package Identity

このドラフトでは、package name は import path の先頭識別子です。

package name の探索場所、version 解決、lock file、package の取得方法はこの仕様では
定義しません。これらは toolchain または後続仕様が定義します。

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

Backend binding は capability 単位ではなく外部関数単位で定義します。たとえば
`platform.io._write_stdout_utf8!` の JavaScript lowering は、その外部関数の `bindings.js` に
定義され、Native runtime symbol call は同じ外部関数の `bindings.native` に定義され
ます。

初期 JavaScript binding は、次の形を持ちます。

```json
{
  "bindings": {
    "js": {
      "symbol": "__emela_write_stdout_utf8"
    }
  }
}
```

`symbol` は backend runtime が提供する JavaScript 関数名です。code generation は
`symbol(arg0, arg1, ...)` へ lower します。`console.log` などの host API 呼び出しは
backend runtime の `.js` file 側に置き、descriptor には直接書きません。

初期 Native binding は、C ABI の runtime symbol call に限定し、次の形を持ちます。

```json
{
  "bindings": {
    "native": {
      "symbol": "emela_write_stdout_utf8",
      "link": ["emela_runtime"]
    }
  }
}
```

`symbol` は C ABI で呼び出す runtime symbol 名です。target 固有の symbol prefix
（macOS の leading `_` など）、ABI、register convention、object format、linker 種別は
Target/Native backend が決定します。`link` は full native build 時に linker へ渡す
runtime library 名の配列です。`--backend native-aarch64-apple-darwin --artifact` のように assembly だけを出力する mode では
link 情報を使用しません。inline syscall や OS 固有命令列への direct lowering はこの
初期 Native binding 形式では定義しません。

外部 declaration は存在するが選択 backend の binding がない場合、code generation は
MUST reject します。

JavaScript backend の初期出力形式は単一 `.js` file です。top-level Emela function は
JavaScript function へ変換されます。executable mode では、`main` または `main!` が
entry point として末尾で呼び出されます。library mode では entry point 呼び出しを
生成せず、関数定義だけを出力してよいです。

### Initial Platform Imports

初期実装は、compiler-known な `platform` package を持ってよいです。この package は
外部 declaration registry として扱われ、通常の package 配布形式や標準ライブラリの
安定 ABI を意味しません。

Public standard library API は `std.*` に置きます。`platform.*` は backend/platform
implementation API であり、標準ライブラリ wrapper からだけ呼ばれる外部実装境界です。
通常の user source は `platform.*` を import してはなりません。

初期の `platform` package は、少なくとも次の外部関数を宣言できます。

| Import path | Type | Capability |
| --- | --- | --- |
| `platform.io._write_stdout_utf8!` | `(String) -> Result<Unit, PlatformError>` | `Stdout` |
| `platform.io._read_stdin_utf8!` | `() -> Result<String, PlatformError>` | `Stdin` |
| `platform.clock._now_i32!` | `() -> I32` | `Clock` |

これらの関数は effectful です。関数名に `!` を持ち、上表の capability requirement を
通常の関数呼び出しと同じ規則で呼び出し元へ伝播します。

Platform primitive の失敗は panic/trap ではなく `Result` 値で返します。
`PlatformError` は少なくとも `Unsupported`, `Unavailable`, `Interrupted`,
`InvalidUtf8`, `Unknown` の variant を持てます。`platform` package の具体的な runtime
symbol、host import name、ABI、時刻の単位はこの仕様では固定しません。

## Examples

標準ライブラリを import して呼び出す例:

```emela
import std.io.write_stdout_utf8!

fn main!() -> Result<Unit, PlatformError> {
  write_stdout_utf8!("hello\n")
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
使ってもよく、将来は backend descriptor、package metadata、または compiled metadata に
置き換えられます。

## Open Questions

- alias import の構文を導入するか。
- import path の区切りに `.` を使い続けるか。
- compiled metadata の形式をどう定義するか。
- 型、struct、enum、trait の import も同じ構文で扱うか。
- import された外部関数の ABI と symbol name をどの仕様で固定するか。
