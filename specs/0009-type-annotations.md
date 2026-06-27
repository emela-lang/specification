# 0009: Type Annotations

Status: Draft

## Summary

このドラフトは、Emela の明示的な型注釈を定義します。

対象は、関数パラメータ、関数戻り値、ローカル束縛です。0009 を採用する実装では、
これらの場所の型注釈は必須です。型注釈は型推論を完全に置き換えるものではなく、
関数境界やローカル束縛の型を明示したうえで、式内部の型を推論・検査するために
使います。

## Motivation

初期の Emela は関数パラメータ型を本体と呼び出し元から推論できます。しかし、その
ままでは小さな例では便利な一方、関数境界が呼び出し関係に依存して変わりやすく
なります。

Emela は関数境界とローカル束縛の型をソース上で明示する方針を取ります。これにより、
次の性質を仕様として扱えます。

- public API か private helper かに関係なく、関数境界がソース上で固定される。
- 呼び出し元だけでは型が決まらない関数パラメータを扱える。
- `Result<I32, Error>` のような generic type の具体的な型引数を指定できる。
- ローカル束縛に期待型を与えて、型エラーを発生箇所に近づけられる。

型注釈を導入することで、型推論の自由度を保ちながら、ソース上の重要な型境界を明示
できます。

## Specification

### Syntax

型注釈は `: type` の形で書きます。

```text
function =
  attribute* fn function_name ( parameter_list? ) return_type block

parameter_list =
  parameter (, parameter)*

parameter =
  identifier type_annotation

return_type =
  -> type

binding =
  identifier type_annotation = expr

type_annotation =
  : type
```

`type` の文法は、現在有効な型仕様の合成です。初期コアの primitive type、struct/enum
の nominal type、generic type application、関数型が含まれます。

```text
type =
  I32
  | Bool
  | Unit
  | type_name
  | type_parameter
  | type_name < type_argument_list >
  | function_type
```

`type_parameter` は、型宣言など、その型パラメータがスコープ内にある場所でのみ有効
です。このドラフトでは generic function を定義しないため、通常の関数パラメータ型
注釈では新しい型パラメータを導入できません。

### Function Parameters

関数パラメータは型注釈を MUST 持ちます。

```emela
fn add(x: I32, y: I32) -> I32 {
  x + y
}
```

パラメータ型注釈が存在しない関数定義は MUST reject されます。

パラメータの型は注釈された型でなければなりません。関数本体内の使用、および呼び
出し時の実引数は、その型と一致しなければなりません。

### Return Types

関数戻り値型注釈は `-> type` の形で、引数リストの後、関数本体の前に書きます。
関数は戻り値型注釈を MUST 持ちます。

```emela
fn is_zero(value: I32) -> Bool {
  value == 0
}
```

関数本体ブロックの型は注釈された型と一致しなければなりません。戻り値型注釈が
存在しない関数定義は MUST reject されます。値を返さない関数は `-> Unit` を明示
します。

### Local Bindings

ローカル束縛は型注釈を MUST 持ちます。

```emela
fn main() -> I32 {
  value: I32 = 42
  value
}
```

右辺式の型は注釈された型と一致しなければなりません。束縛型注釈が存在しない
ローカル束縛は MUST reject されます。

ローカル束縛の型注釈は新しい型を導入しません。未定義の型名、型引数の個数が一致
しない generic type、スコープ外の型パラメータを含む注釈は MUST reject されます。

### Generic Type Uses

型注釈では generic type application を書けます。

```emela
struct Error {
  code: I32
}

enum Result<T, E> {
  Ok(T)
  Err(E)
}

fn unwrap_or_zero(result: Result<I32, Error>) -> I32 {
  match result {
    Ok(value) -> value
    Err(_) -> 0
  }
}
```

generic type を注釈に使う場合、型引数の個数は宣言された型パラメータの個数と一致
しなければなりません。型引数を省略した generic type の使用は MUST reject されます。

```emela
-- invalid: Result requires two type arguments
fn unwrap_or_zero(result: Result) -> I32 {
  0
}
```

このドラフトは generic function を定義しません。そのため、次のように関数が独自の
型パラメータを導入する構文は無効です。

```emela
-- invalid in this draft
fn identity<T>(value: T) -> T {
  value
}
```

### Type Checking

型注釈は期待型として扱います。

- `parameter: T` は、そのパラメータの型を `T` に固定します。
- `fn f(...) -> T` は、関数本体ブロックの型を `T` に制約します。
- `name: T = expr` は、`expr` の型を `T` に制約し、`name` を `T` として導入します。
- これらの注釈を省略した関数定義またはローカル束縛は reject されます。

型注釈によって暗黙の型変換は発生しません。推論された型と注釈された型が一致しない
場合、プログラムは MUST reject されます。

型注釈は effect や capability を消去しません。注釈された式または関数本体が effectful
である場合、effect system と platform capability の規則は通常通り適用されます。

## Examples

関数境界をすべて注釈する例:

```emela
fn max(left: I32, right: I32) -> I32 {
  match left < right {
    true -> right
    false -> left
  }
}
```

ローカル束縛へ期待型を与える例:

```emela
fn main() -> Bool {
  value: I32 = 42
  value == 42
}
```

generic enum の具体型を関数パラメータで指定する例:

```emela
enum Option<T> {
  Some(T)
  None
}

fn is_some(value: Option<I32>) -> Bool {
  match value {
    Some(_) -> true
    None -> false
  }
}
```

関数型をパラメータとローカル束縛で指定する例:

```emela
fn add_one(value: I32) -> I32 {
  value + 1
}

fn apply(value: I32, f: fn(I32) -> I32) -> I32 {
  f(value)
}

fn main() -> I32 {
  op: fn(I32) -> I32 = add_one
  apply(41, op)
}
```

## Compilation Notes

型注釈は type checker の制約として扱えます。実装は、注釈された型を fresh type variable
へ unify してもよく、関数シグネチャやローカル環境へ直接登録してもよいです。

generic type application を含む注釈は、型検査時に宣言の arity と型引数を検証します。
後続の lowering では、具体化された型を monomorphization の入力として使えます。

このドラフトは runtime representation や ABI を変更しません。

## Open Questions

- 型注釈を必須にする境界を、public API や package export で導入するか。
- `let` または `val` など、束縛キーワードを導入した場合の注釈位置。
- pattern binding に型注釈を許すか。
- `as` などの明示的な型変換構文を別途導入するか。
- generic function と trait bound の構文。
