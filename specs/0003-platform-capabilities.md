# 0003: Platform Capabilities

Status: Draft

## Summary

このドラフトは、Native/WASM などの target が提供する platform-dependent な機能を
Effect System の一部として扱うための仕様です。

Emela では、ファイル、標準入出力、時計、乱数、環境変数、プロセス、ネットワーク、
WASM host import などを暗黙のグローバル機能として扱いません。これらは platform
capability として表現し、effect と同じく静的に追跡します。

## Motivation

マルチプラットフォームな Native ビルドと WASM ビルドを同じ言語仕様で扱うには、
OS や host が提供する機能をプログラムの意味から分離する必要があります。

platform capability を Effect System に吸収すると、次の性質を仕様として扱えます。

- portable な pure core と target-specific な境界を分離できる。
- `File` や `Network` が存在しない target では、型検査または capability 検査で reject
  できる。
- WASM host import と Native syscall/runtime call を同じ抽象境界で扱える。
- テストでは capability を差し替え、実行時には target が提供する実装へ接続できる。

## Specification

### Capability

capability は、target または host が提供する外部機能を表します。

初期ドラフトでは、少なくとも次の capability kind を予約します。

| Capability | 意味 |
| --- | --- |
| `Stdout` | 標準出力への書き込み |
| `Stdin` | 標準入力からの読み込み |
| `Stderr` | 標準エラー出力への書き込み |
| `FileRead` | ファイルシステムからの読み込み |
| `FileWrite` | ファイルシステムへの書き込み |
| `Clock` | 現在時刻または monotonic time の取得 |
| `Random` | 外部乱数源の利用 |
| `Env` | 環境変数または実行環境設定の読み取り |
| `Process` | プロセス終了、引数、子プロセスなどの操作 |
| `Network` | ネットワーク I/O |
| `HostImport` | WASM host import または embedder 提供関数の呼び出し |

capability は effect の詳細化です。たとえば `Stdout` を要求する式は `IO` effect と
`Platform` effect を持ちます。

### Requires Attribute

source code 上で capability requirement を明示する場合、関数定義の直前に
`#[requires(...)]` 属性を書きます。

```emela
#[requires(Stdout)]
fn print_i32!(value) {
  ()
}
```

`#[requires(Stdout)]` は、その関数が `Stdout` capability を要求することを表します。

複数の capability を要求する場合は、カンマ区切りで列挙します。

```emela
#[requires(FileRead, Env)]
fn load_config!() {
  read_config!()
}
```

`#[requires(...)]` は関数の capability signature を明示する属性です。関数型としては、
effect row に加えて capability row を持つものとして扱います。

```text
print_i32!: (I32) -> Unit !{IO, Platform} requires {Stdout}
```

上の型は、`Stdout` capability を要求し、I/O と platform-dependent な振る舞いを
発生させうる関数を表します。

### Inference

通常の関数では、capability requirement は本体から推論できます。

```emela
fn main!() {
  print_i32!(42)
}
```

`print_i32!` が `Stdout` を要求する場合、`main!` も `Stdout` を要求すると推論されます。

ただし、次の場所では `#[requires(...)]` による明示を MUST 要求します。

- 外部関数または host import の宣言。
- platform runtime が提供する primitive 関数。

次の場所では `#[requires(...)]` による明示を SHOULD 使用します。

- public API として capability boundary を固定したい関数。
- 推論された capability set より狭い、または明示的に検査したい capability set を持つ
  関数。

`#[requires(...)]` が書かれた関数では、推論された capability set が属性に列挙された
capability set に収まらなければ MUST reject されます。

`#[requires()]` は、capability を要求しないことを明示します。これは pure/portable な
API 境界を固定したい場合に使えます。

### Propagation

capability を要求する関数を呼び出す式は、同じ capability を要求します。

```emela
#[requires(Stdout)]
fn print_i32!(value) {
  ()
}

fn main!() {
  print_i32!(42)
}
```

`print_i32!` が `Stdout` を要求する場合、`main!` も `Stdout` を要求します。ただし、
`main!` が capability handler または runtime boundary で `Stdout` を処理する場合は、
外側へ伝播しません。

### Platform Capability Set

Platform は、提供できる capability set を宣言します。

この仕様でいう Platform は、target triple そのものではありません。Platform は、
capability requirement を満たす外部実装セットです。target triple は native ABI、
object format、assembly backend、linker などを選ぶための概念として残ります。
言い換えると、Target は ABI/codegen/linker の責務を持ち、Platform は capability と
外部関数実装 binding の provider です。

同じ target triple に対して複数の Platform を選べます。たとえば `node` Platform は
JavaScript backend の `console.log` binding を提供でき、`wasm32-wasi` target の既定
Platform は WASI import を提供できます。

Platform は build input または manifest として compiler へ渡されます。manifest は
少なくとも platform 名、提供 capability set、外部関数 declaration を含みます。

```json
{
  "name": "node",
  "capabilities": ["Stdout", "Clock"],
  "externs": [
    {
      "path": ["platform", "io"],
      "name": "print_i32!",
      "params": ["I32"],
      "return": "Unit",
      "effectful": true,
      "capabilities": ["Stdout"],
      "bindings": {
        "js": {
          "callee": "console.log"
        },
        "native": {
          "symbol": "emela_print_i32",
          "link": ["emela_runtime"]
        }
      }
    }
  ]
}
```

`capabilities` に含まれない capability を runtime boundary が要求する場合、その
Platform への compilation は MUST reject されます。

外部関数ごとの `capabilities` は「その関数を呼ぶために必要な capability」を表します。
Platform は capability を提供するだけでなく、選択 backend に必要な binding も該当
外部関数ごとに提供しなければ code generation できません。

#### Default Target Platforms

実装は、従来の target triple に対応する既定 Platform を持ってよいです。既定
Platform の capability set は次のように定義できます。

```text
platform aarch64-apple-darwin provides {
  Stdout,
  Stdin,
  Stderr,
  FileRead,
  FileWrite,
  Clock,
  Random,
  Env,
  Process,
  Network
}

platform x86_64-unknown-linux-gnu provides {
  Stdout,
  Stdin,
  Stderr,
  FileRead,
  FileWrite,
  Clock,
  Random,
  Env,
  Process,
  Network
}

platform wasm32-unknown-unknown provides {
}

platform wasm32-wasi provides {
  Stdout,
  Stdin,
  Stderr,
  FileRead,
  FileWrite,
  Clock,
  Random,
  Env
}
```

Platform が提供しない capability を未処理のまま要求するプログラムは、その Platform へ
MUST compile できません。

### Runtime Boundary

executable mode のプログラムでは、`main` または `main!` が runtime boundary です。
library mode の compilation unit は runtime boundary を選択しません。library mode
では、各関数の effect/capability requirement は型情報として保持され、最終的な
platform capability 提供チェックは executable mode または link/package composition
の段階で行われます。

runtime boundary は、Platform capability set とプログラムが要求する capability set を
照合します。

Platform が capability を提供する場合、コンパイラまたは runtime はその capability を
具体的な実装へ lower できます。Platform が提供しない場合、プログラムは reject される
か、ユーザーが明示的な handler/import を提供する必要があります。

### Portable Functions

capability を要求しない関数は portable です。

portable な関数は、target に依存せず同じ意味を持つ必要があります。Native と WASM の
どちらでも、同じ入力に対して同じ値と同じ pure/effect behavior を持たなければなり
ません。

### Host Imports

WASM target では、host import は `HostImport` capability として扱います。

host import の名前、型、effect、capability requirement は静的に宣言される必要が
あります。未宣言の host import を暗黙に呼び出してはなりません。

## Examples

標準出力を要求する関数:

```emela
#[requires(Stdout)]
fn print_i32!(value) {
  ()
}
```

portable な関数:

```emela
fn add_one(x) {
  x + 1
}
```

target が `Stdout` を提供する場合だけ compile できる関数:

```emela
fn main!() {
  print_i32!(42)
}
```

## Compilation Notes

Native target では、capability は runtime capability value、直接 syscall/libc call、
または platform runtime 関数へ lower できます。初期 Native manifest binding は
外部関数単位の C ABI runtime symbol call を表し、symbol prefix、ABI、register
convention、linker 種別は Target/Native backend 側で処理します。

WASM target では、capability は WASI import、custom host import、または embedder が
提供する function table へ lower できます。

capability を明示的な値として渡す実装にすると、テスト時に mock capability を注入
できます。ただし、言語仕様として capability が値であることを要求するかは未決定です。

## Open Questions

- capability row を effect row に統合するか、`#[requires(...)]` として分離するか。
- `main!` を entry point として許可するか、`main` に特別な runtime boundary を与えるか。
- capability handler の構文をどうするか。
- `IO` effect と `Platform` effect を分け続けるか、capability だけで十分か。
- Platform capability set をソース内に書くか、ビルド設定側で宣言するか。
- WASM の `wasi` と custom host import を同じ capability model で扱うか。
- `#[requires()]` を空 capability set の明示として許可するか、それとも属性自体を省略
  させるか。
