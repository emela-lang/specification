# 0011: Pipeline Syntax

Status: Draft

## Summary

このドラフトは、Emela の forward pipeline 構文 `|>` を定義します。

Pipeline は値変換の流れを左から右へ書くための構文糖衣です。初期仕様では、pipeline
stage は明示的な関数呼び出しに限定し、左辺の値を stage call の第 1 引数として渡し
ます。

```emela
"hello" |> write_stdout_utf8!()
```

は次と同じ意味です。

```emela
write_stdout_utf8!("hello")
```

このドラフトは placeholder、partial application、Result-aware pipe、qualified
function call を導入しません。

## Motivation

Package manager、stdlib、manifest parser、dependency resolver では、値を段階的に
変換する処理が多く現れます。

```emela
text = read_text!(path)?
manifest = parse_manifest(text)?
plan = resolve(manifest)?
execute!(plan)?
```

Pipeline 構文により、中間変数を減らし、データの流れをソース上で明示できます。

```emela
path
  |> read_text!()?
  |> parse_manifest()?
  |> resolve()?
  |> execute!()?
```

Emela は module function と effect/capability を重視します。Pipeline は method を
各型へ大量に追加せず、module function を組み合わせて読みやすく書くための構文です。

## Specification

### Syntax

Pipeline expression は、通常の式の後ろに 1 個以上の pipeline stage を続けた式です。

```text
expr =
  pipe_expr

pipe_expr =
  comparison_expr ( "|>" pipe_stage )*

pipe_stage =
  function_name "(" argument_list? ")" postfix_operator*

function_name =
  identifier !?

postfix_operator =
  "?"

argument_list =
  expr ( "," expr )*
```

`function_name` は、通常の関数呼び出しと同じ名前解決規則を使います。

初期仕様では、pipeline stage は必ず `f(...)` の形を持ちます。`value |> f` のように
関数名だけを書く構文は定義しません。これは関数値参照と関数呼び出しを曖昧にしない
ためです。

### Desugaring

Pipeline は型検査前に通常の関数呼び出しへ desugar できます。

```emela
left |> f()
```

は次へ desugar します。

```emela
f(left)
```

```emela
left |> f(a, b)
```

は次へ desugar します。

```emela
f(left, a, b)
```

複数 stage は左結合です。

```emela
x |> f() |> g(y)
```

は次と同じ意味です。

```emela
g(f(x), y)
```

### Precedence

Pipeline operator は comparison expression より弱く、binding `=` より強く結合します。

```emela
x + y |> f()
```

は次と同じ意味です。

```emela
(x + y) |> f()
```

```emela
x |> f() == y
```

は次と同じ意味です。

```emela
(x |> f()) == y
```

### Effects and Capabilities

Pipeline 自体は新しい effect や capability を導入しません。Desugar 後の関数呼び出し
が通常の effect/capability 規則で検査されます。

```emela
fn main!() -> Unit {
  "hello" |> write_stdout_utf8!()
}
```

上の `write_stdout_utf8!` が `Stdout` capability を要求する場合、`main!` も通常の関数呼び出し
と同じ規則で `Stdout` を要求します。

### Error Propagation

`?` は pipeline stage call 全体に後置できます。`?` の意味は Result/Option ergonomics
を定義する後続仕様に従います。この仕様では、pipeline と `?` の結合だけを定めます。

```emela
path |> read_text!()?
```

は次と同じように扱われます。

```emela
read_text!(path)?
```

`?` の実際の型規則と lowering はこの仕様では定義しません。

### Out of Scope

初期 Pipeline 構文では次を定義しません。

- `value |> f` のような関数値 stage。
- `_` placeholder による任意引数位置への挿入。
- `|?>` などの Result-aware pipe operator。
- `std.path.join()` のような qualified function call stage。
- partial application。
- async/stream pipeline semantics。

## Examples

標準ライブラリ関数への pipe:

```emela
import std.io.write_stdout_utf8!

fn main!() -> Unit {
  "hello" |> write_stdout_utf8!()
}
```

値変換:

```emela
fn add_one(value: I32) -> I32 {
  value + 1
}

fn double(value: I32) -> I32 {
  value * 2
}

fn main() -> I32 {
  20
    |> add_one()
    |> double()
}
```

引数付き stage:

```emela
fn add(value: I32, by: I32) -> I32 {
  value + by
}

fn main() -> I32 {
  20 |> add(22)
}
```

## Compilation Notes

実装は parser または early lowering pass で pipeline を通常の `Expr::Call` へ変換して
よいです。その場合、type checker、effect checker、native backend、JavaScript backend
は pipeline 専用の処理を持つ必要がありません。

Desugar 後の source location を保持できる実装では、型エラーや arity error を pipeline
stage の位置に報告するべきです。

## Open Questions

- `value |> f` を関数値呼び出しとして許すか。
- `_` placeholder を導入するか。導入する場合、複数 placeholder を許すか。
- Qualified function call を module system と同じ仕様で定義するか。
- `?` と pipeline の詳細な型規則を Result/Option ergonomics 仕様でどう扱うか。
- Pipeline operator を pattern matching や future async syntax とどう組み合わせるか。
