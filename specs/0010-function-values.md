# 0010: Function Values and Function Types

Status: Draft

## Summary

このドラフトは、Emela で関数を値として扱うための仕様です。

トップレベル関数と import された外部関数は、関数値としてローカル束縛へ代入でき、
関数パラメータとして渡せます。これにより、関数を受け取る関数、関数を選択して
束縛するコード、callback-style の API を表現できます。

このドラフトは lambda expression、closure capture、partial application を導入しません。
最初の対象は、名前を持つ関数を値として参照し、明示的な関数型注釈で検査する範囲です。
関数を返す関数は、戻り値型に関数型を書くことで表現できます。

## Motivation

Emela では 0009 により関数境界とローカル束縛の型注釈が必須になります。関数を値と
して渡したい場合、その値を受け入れる型構文が必要です。

関数値を導入すると、次の性質を仕様として扱えます。

- 同じ型の処理を高階関数へ渡せる。
- callback を明示的な effect marker とともに表現できる。
- runtime や platform import が提供する関数を、通常の値と同じスコープ規則で扱える。
- lambda と closure を導入する前に、関数型・呼び出し・effect の規則を固定できる。

## Specification

### Function Type Syntax

関数型は `fn` type constructor で書きます。

```text
type =
  ...
  | function_type

function_type =
  fn effect_marker? ( type_list? ) -> type

effect_marker =
  !

type_list =
  type (, type)*
```

`fn(I32, I32) -> I32` は、`I32` を 2 つ受け取り `I32` を返す pure な関数型です。

`fn!(I32) -> Unit` は、`I32` を 1 つ受け取り `Unit` を返す effectful な関数型です。
`!` は関数名に付く effect marker と同じ意味を持ちます。

引数を取らない関数型は `fn() -> T` と書きます。

```emela
value: fn() -> I32 = read_cached_value
```

関数型の戻り値型も通常の `type` です。そのため、関数を返す関数は
`fn make() -> fn(I32) -> I32` のように書けます。

関数型は通常の `type` なので、generic type の型引数としても使えます。

```emela
struct Handler<T> {
  callback: T
}

fn add_one(value: I32) -> I32 {
  value + 1
}

fn main() -> I32 {
  handler: Handler<fn(I32) -> I32> = Handler { callback: add_one }
  handler.callback(41)
}
```

このドラフトでは、関数型に capability set を直接書く構文は定義しません。effectful
な関数値の capability requirement を型に含めるか、別の effect/capability row として
扱うかは後続仕様で定義します。

### Function Values

トップレベル関数名は、値が期待される位置で関数値として参照できます。

```emela
fn add(x: I32, y: I32) -> I32 {
  x + y
}

fn main() -> I32 {
  op: fn(I32, I32) -> I32 = add
  op(20, 22)
}
```

import された外部関数名も、解決済みの function signature が関数型と一致する場合、
関数値として参照できます。

```emela
import platform.io.print_i32!

fn call_stdout!(value: I32, callback: fn!(I32) -> Unit) -> Unit {
  callback(value)
}
```

関数値は immutable value です。関数値の比較、直列化、数値への変換、実行時の identity
観測はこのドラフトでは定義しません。

### Function Parameters

関数パラメータは関数型注釈を持てます。

```emela
fn apply(value: I32, transform: fn(I32) -> I32) -> I32 {
  transform(value)
}

fn add_one(value: I32) -> I32 {
  value + 1
}

fn main() -> I32 {
  apply(41, add_one)
}
```

実引数として渡される関数値は、パラメータの関数型と arity、引数型、戻り値型、
effect marker が一致しなければなりません。

pure な関数型 `fn(...) -> T` が期待される場所に effectful な関数値を渡すことは
MUST reject されます。

effectful な関数型 `fn!(...) -> T` が期待される場所には effectful な関数値を渡せます。
pure な関数値を `fn!(...) -> T` として扱うことを許すかどうかは未決定です。この
ドラフトでは実装はどちらを選んでもよいですが、選択は一貫していなければなりません。

### Function Return Values

関数は関数型を返せます。

```emela
fn add_one(value: I32) -> I32 {
  value + 1
}

fn identity(value: I32) -> I32 {
  value
}

fn select(flag: Bool) -> fn(I32) -> I32 {
  match flag {
    true -> add_one
    false -> identity
  }
}
```

戻り値式の型は、戻り値注釈の関数型と一致しなければなりません。複数の branch が
関数値を返す場合、各 branch の関数型は同じでなければなりません。

### Local Bindings

ローカル束縛は関数型注釈を持てます。

```emela
fn sub(left: I32, right: I32) -> I32 {
  left - right
}

fn main() -> I32 {
  op: fn(I32, I32) -> I32 = sub
  op(50, 8)
}
```

0009 と同じく、ローカル束縛の型注釈は必須です。関数値を束縛する場合も、束縛型
注釈は関数型でなければなりません。

### Calls Through Function Values

関数型を持つ式は、通常の関数呼び出しと同じ call syntax で呼び出せます。

```emela
callback(value)
```

callee の型が関数型でない場合、呼び出しは MUST reject されます。

呼び出し時の実引数の数と型は、関数型のパラメータ型と一致しなければなりません。
呼び出し式の型は関数型の戻り値型です。

effectful な関数値を呼び出す式は effectful です。pure な関数内で effectful な関数
値を呼び出すことは、通常の effect rule と同じく MUST reject されます。

### Name Resolution

識別子が値位置に現れる場合、次の順で解決します。

1. ローカル束縛または関数パラメータ。
2. payload を持たない enum variant。
3. トップレベル関数または import された外部関数。

同じ名前が複数の同一優先度候補へ解決される場合は MUST reject されます。

関数呼び出し `name(args...)` は、既存の direct call として解決してもよく、
`name` を関数値へ解決してから call-through-function-value として検査してもよいです。
どちらの実装戦略でも、観測可能な型検査結果と effect/capability propagation は同じで
なければなりません。

### Type Checking

トップレベル関数は、次のような関数型を持ちます。

```text
fn add(x: I32, y: I32) -> I32
  has type fn(I32, I32) -> I32

fn print_i32!(value: I32) -> Unit
  has type fn!(I32) -> Unit
```

関数値の型は、その関数の declared parameter types、return type、effect marker から
構築します。

import された外部関数の型は、外部 declaration の parameter types、return type、
effect information から構築します。

関数値を束縛または引数として渡す場合、期待される関数型と実際の関数型が一致しなければ
なりません。関数型は structural type として比較します。つまり、同じ名前の関数である
必要はなく、arity、各引数型、戻り値型、effect marker が一致すれば同じ関数型です。

### Capabilities

このドラフトでは、関数型は capability requirement を直接含みません。

effectful な関数値の呼び出しによってどの capability が要求されるかは、実装が関数値の
由来を追跡できる場合は元の関数の capability requirement を伝播してよいです。

関数値がパラメータとして渡される場合、callee 側で必要な capability set を静的に
決定できない実装は、保守的にその呼び出しを reject してもよいです。関数型に capability
row を含める構文は後続仕様で定義します。

## Examples

関数を引数として受け取る例:

```emela
fn apply(value: I32, f: fn(I32) -> I32) -> I32 {
  f(value)
}

fn double(value: I32) -> I32 {
  value * 2
}

fn main() -> I32 {
  apply(21, double)
}
```

関数をローカル束縛に入れる例:

```emela
fn add(left: I32, right: I32) -> I32 {
  left + right
}

fn main() -> I32 {
  op: fn(I32, I32) -> I32 = add
  op(20, 22)
}
```

関数を返す例:

```emela
fn double(value: I32) -> I32 {
  value * 2
}

fn identity(value: I32) -> I32 {
  value
}

fn choose_transform(flag: Bool) -> fn(I32) -> I32 {
  match flag {
    true -> double
    false -> identity
  }
}

fn main() -> I32 {
  transform: fn(I32) -> I32 = choose_transform(true)
  transform(21)
}
```

effectful な callback を受け取る例:

```emela
import platform.io.print_i32!

fn each_pair!(left: I32, right: I32, callback: fn!(I32) -> Unit) -> Unit {
  callback(left)
  callback(right)
}

fn main!() -> Unit {
  each_pair!(20, 22, print_i32!)
}
```

## Compilation Notes

関数値は、実装上は statically known function pointer、symbol reference、または
compiler-internal function id として lower できます。

このドラフトは closure environment を定義しないため、関数値は捕捉環境を持ちません。
lambda や closure を後続仕様で導入する場合、関数型を再利用しつつ、runtime representation
に環境 pointer を追加できます。

native backend は、direct call と indirect call を分けて lower できます。関数値の
callee が静的に既知であれば direct call へ最適化してもよいです。

## Open Questions

- pure な関数値を `fn!(...) -> T` として扱う subtyping を許すか。
- function type に capability row を書く構文。
- lambda expression と closure capture の構文。
- partial application または currying を導入するか。
- function pointer representation と ABI を固定するか。
