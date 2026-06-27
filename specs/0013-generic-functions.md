# 0013: Generic Functions

Status: Draft

## Summary

このドラフトは、型パラメータを持つ top-level function を定義します。

generic function は呼び出しごとに型引数を推論、または明示的な型引数で instantiate
されます。v1 では trait bound、`where` clause、generic function value は定義しません。

## Motivation

`id`、`map`、`unwrap_or` のような関数は、同じ本体を複数の型で使える必要があります。
generic type だけでは、payload を保持する型は抽象化できますが、それらを操作する関数を
型ごとに重複して定義する必要が残ります。

## Specification

function declaration は function name の直後に型パラメータリストを持てます。

```text
function =
  fn function_name type_parameter_list? ( function_param_list? ) return_annotation block

type_parameter_list =
  < type_parameter (, type_parameter)* >

type_parameter =
  identifier
```

```emela
fn id<T>(value: T) -> T {
  value
}
```

型パラメータ名は、同じ function の型パラメータリスト内で一意でなければなりません。
型パラメータは、その function の parameter type、return type、local type annotation、
および nested type argument 内で使用できます。

型パラメータは function body の外側へスコープしません。

`main` と `main!` は generic function であってはなりません。

### Calls

generic function call は、通常の function call と同じ構文、または function name の直後に
型引数リストを置く構文を使います。

```text
call_expr =
  function_name type_argument_list? ( argument_list? )

type_argument_list =
  < type (, type)* >
```

```emela
fn id<T>(value: T) -> T {
  value
}

fn main() -> Bool {
  first: I32 = id(42)
  id(true)
}
```

呼び出し時、toolchain は argument type と parameter type から型引数を推論します。
同じ型パラメータが複数箇所に現れる場合、すべて同じ concrete type に解決されなければ
なりません。

```emela
fn choose<T>(left: T, right: T) -> T {
  left
}

fn ok() -> I32 {
  choose(1, 2)
}

-- invalid: T cannot be both I32 and Bool
fn bad() -> I32 {
  choose(1, true)
}
```

型引数が argument type だけで決まらない場合、return type の利用文脈や explicit local
annotation によって決まってよいです。最終的に型が決まらない値は reject されます。

明示的な型引数は、function の型パラメータと同じ個数でなければなりません。2 個以上の
型引数も同じ規則で使用できます。

```emela
fn ok<T, E>(value: T) -> Result<T, E> {
  Ok(value)
}

fn main() -> Result<I32, PlatformError> {
  ok<I32, PlatformError>(42)
}
```

明示的な型引数と argument type から推論される型引数が矛盾する場合、toolchain は reject
しなければなりません。

```emela
fn id<T>(value: T) -> T {
  value
}

-- invalid: T is explicitly Bool, but argument has type I32
fn bad() -> I32 {
  id<Bool>(42)
}
```

### Body Checking

generic function body は、型パラメータを rigid type として検査します。trait bound が
まだないため、型パラメータに対して concrete type 固有の operator や method を使っては
なりません。

```emela
-- invalid in v1: T is not known to implement +
fn add_one<T>(value: T) -> T {
  value + 1
}
```

型パラメータ同士の同一性は使えます。

```emela
fn pair_first<T>(left: T, right: T) -> T {
  left
}
```

### Function Values

v1 では generic function を first-class function value として扱いません。

```emela
fn id<T>(value: T) -> T {
  value
}

-- invalid in v1
fn main() -> Unit {
  f: fn(I32) -> I32 = id
  ()
}
```

monomorphic function value と function type は [0010](0010-function-values.md) の規則に
従います。

## Examples

generic function と generic enum を組み合わせる例:

```emela
fn ok<T, E>(value: T) -> Result<T, E> {
  Ok(value)
}

fn main() -> Result<I32, PlatformError> {
  ok(42)
}
```

## Compilation Notes

実装は generic function を呼び出しごとに monomorphize してよいです。dynamic backend は
同じ JavaScript function を複数の instantiation で共有してもよいです。

native backend や static backend は、使用された型引数の組ごとに concrete signature を
生成できます。v1 は具体化後の symbol naming scheme や ABI を定義しません。

## Open Questions

- trait bound と `where` clause の構文。
- generic function value をどのように型付けするか。
- recursive generic function の monomorphization 制限。
