# 0002: Traits and Operators

Status: Draft

## Summary

このドラフトは、Emela の trait と二項演算子の関係を定義します。

二項演算子は組み込み関数ではなく、trait method call への糖衣構文です。たとえば
`x + y` は、`x` の型が `Add` trait を実装している場合に `x.add(y)` として扱われます。

## Motivation

演算子を primitive に固定すると、ユーザー定義型に対する加算、比較、等価性を後から
自然に拡張しにくくなります。

演算子を trait に結びつけることで、次の性質を得ます。

- `+` などの記法を primitive 型とユーザー定義型の両方で使える。
- 演算子の意味を型クラス的な制約として静的に検査できる。
- Native/WASM lowering では、primitive 実装だけを特別扱いし、言語意味は trait に
  保てる。

## Specification

### Trait

trait は、型が提供すべき method set を定義します。

このドラフトでは trait 宣言の完全な構文は確定しません。概念上、`Add` は次の
method を要求します。

```text
trait Add<Rhs = Self> {
  add(self, rhs: Rhs) -> Output
}
```

`Self` は実装対象の型を表します。

`Output` は method ごとの関連型として扱うか、trait parameter として扱うか未決定です。

### Method Call

method call は次の形で書きます。

```emela
receiver.method(arg1, arg2)
```

method call の探索は、`receiver` の型に実装された trait method を対象にします。

同じ名前の method が複数候補になる場合の解決規則は未決定です。

### Operator Desugaring

二項演算子は、型検査前または型検査中に trait method call として解釈されます。

| Operator | Desugared form | Required trait |
| --- | --- | --- |
| `x + y` | `x.add(y)` | `Add` |
| `x - y` | `x.sub(y)` | `Sub` |
| `x * y` | `x.mul(y)` | `Mul` |
| `x == y` | `x.eq(y)` | `Eq` |
| `x < y` | `x.lt(y)` | `Ord` |

`x + y` は、`x` の型が `Add` を実装していない場合 MUST reject されます。

desugaring は evaluation order を変更してはなりません。`x + y` では、strict
evaluation に従って `x`、`y` の順に評価してから `add` を呼び出します。

### Primitive Implementations

初期環境では、`I32` が次の trait を実装します。

| Type | Trait | Method behavior |
| --- | --- | --- |
| `I32` | `Add` | two's-complement wrapping addition |
| `I32` | `Sub` | two's-complement wrapping subtraction |
| `I32` | `Mul` | two's-complement wrapping multiplication |
| `I32` | `Eq` | integer equality |
| `I32` | `Ord` | signed integer ordering |

`Bool` は `Eq` を実装します。

### Effects

operator desugaring により呼ばれる method は、通常の関数呼び出しと同じ effect rule に
従います。

pure な関数内で `x + y` を使えるのは、解決された `add` method が pure な場合だけです。

`add!` のような effectful method を演算子から呼び出せるかは未決定です。

## Examples

`I32` の加算:

```emela
fn add_one(x) {
  x + 1
}
```

概念的には次と同じです。

```emela
fn add_one(x) {
  x.add(1)
}
```

`Add` を実装していない型に `+` を使うプログラムは reject されます。

## Compilation Notes

primitive 型に対する trait method は、実装が直接ターゲット命令へ lower してもよい
です。たとえば `I32.add` は WASM の `i32.add` に lower できます。

ユーザー定義型に対する trait method は、通常の関数呼び出しまたは静的 dispatch へ
lower できます。

dynamic dispatch を導入するかどうかは、このドラフトでは定義しません。

## Open Questions

- trait 宣言と impl 宣言の具体的な構文。
- `Output` を関連型にするか、trait parameter にするか。
- method の名前衝突をどう解決するか。
- operator method は必ず pure であるべきか。effectful operator を許可するべきか。
- `==` の戻り値は常に `Bool` に固定するか。
- `Ord` を `<` だけの trait にするか、比較全体を表す trait にするか。
