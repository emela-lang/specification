# 0007: Generic Types

Status: Draft

## Summary

このドラフトは、Emela に型パラメータを持つ `struct` と `enum` を導入します。
最初の対象は、型宣言と型注釈で使う generic type です。generic function、trait bound、
型推論の詳細は後続仕様で定義します。

## Motivation

`Result` や `Option` のようなデータ型は、payload 型だけが異なる同じ構造を繰り返し
定義せずに使える必要があります。

```emela
enum Result<T, E> {
  Ok(T)
  Err(E)
}
```

型パラメータを導入すると、成功値と失敗値の型を用途ごとに変えながら、同じ enum 定義を
再利用できます。

## Specification

`struct` と `enum` は型パラメータリストを持てます。

```text
struct_decl =
  struct type_name type_parameter_list? { struct_field }

enum_decl =
  enum type_name type_parameter_list? { enum_variant* }

type_parameter_list =
  < type_parameter (, type_parameter)* >

type_parameter =
  identifier

type =
  I32
  | Bool
  | Unit
  | type_name
  | type_parameter
  | type_name < type_argument_list >

type_argument_list =
  type (, type)*
```

型パラメータは、その `struct` または `enum` 宣言のフィールド型、variant payload 型、
および同じ型宣言内の nested type argument で使用できます。

後続仕様で追加される型構文も `type` に含まれる場合、型引数として使用できます。
たとえば [0010: Function Values and Function Types](0010-function-values.md) の関数型は、
`type_argument_list` 内に書けます。

```emela
struct Handler<T> {
  callback: T
}

fn add_one(value: I32) -> I32 {
  value + 1
}

fn main() -> Unit {
  handler: Handler<fn(I32) -> I32> = Handler { callback: add_one }
  ()
}
```

```emela
struct Box<T> {
  value: T
}

enum Result<T, E> {
  Ok(T)
  Err(E)
}
```

型適用は `Name<T, E>` の形で書きます。型引数の個数は、宣言された型パラメータの個数と
一致しなければなりません。

```emela
struct Error {
  code: I32
}

fn parse(value) -> Result<I32, Error> {
  match value == 0 {
    true -> Err(Error { code: 1 })
    false -> Ok(value)
  }
}
```

型パラメータ名は、同じ型パラメータリスト内で一意でなければなりません。
型パラメータは、その宣言の外側へスコープしません。

generic type は nominal type です。たとえば `Result<I32, Error>` と
`Result<Bool, Error>` は異なる型として扱います。

このドラフトでは、型引数を省略した generic type の使用は無効です。

```emela
-- invalid: Result requires two type arguments
fn parse(value) -> Result {
  Ok(value)
}
```

## Type Checking

generic type の constructor と pattern は、型適用後の payload 型で検査します。

`Result<I32, Error>` に対する `Ok(value)` は `value: I32` を要求します。
`Err(error)` は `error: Error` を要求します。

`match` の scrutinee が generic enum の型適用である場合、各 variant pattern の payload
binding は型引数を代入した後の型を持ちます。

```emela
fn unwrap_or_zero(result) -> I32 {
  match result {
    Ok(value) -> value
    Err(_) -> 0
  }
}
```

このドラフトでは、関数パラメータの型注釈や generic function をまだ定義しません。
したがって上の例の `result` 型をどのように推論・注釈するかは後続仕様で定義します。

## Compilation Notes

generic type は monomorphization で lower できます。実装は、使用された型引数の組ごとに
具体化された型 layout を生成してよいです。

単一フィールド generic struct は、型引数を代入したフィールド型と同じ表現へ lower できます。
generic enum は、型引数を代入した各 payload 型に基づいて layout を決定できます。

このドラフトは安定 ABI を定義しません。

## Open Questions

- generic function と関数パラメータ型注釈の構文。
- trait bound や `where` clause の構文。
- higher-kinded type を将来導入するかどうか。
- generic type の明示的な type alias。
- monomorphization 以外の実装戦略を許すかどうか。
