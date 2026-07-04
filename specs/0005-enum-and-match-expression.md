## 0005: Enum and match expression

Status: Accepted

`Enum` と `match`， exhaustiveness check を定義する．

### Enum

```emela
enum Either<L, R> {
    Left(L)
    Right(R)
}
```

```emela
enum Color {
    Red
    Green
    Blue
}
```

### Variant へのアクセス

enum の variant は，enum 名で修飾した**型パス** `Enum::Variant` で参照・構築する．
区切りは `.`（ドット）ではなく `::`（二重コロン）である．

```emela
let x = Color::Red
let y = Either::Left(1)
```

- `::` の左辺は enum 型名でなければならない．`.`（ドット）は値のメンバ／モジュール修飾呼び出し
  （spec 0018）に予約され，variant アクセスには用いない．`Color.Red` は error である．
- payload のない variant（`Color::Red`）はそれ自体が値である．payload を持つ variant
  （`Either::Left(1)`）は引数を伴って構築する．
- 型パス `Enum::Variant` の解決規則は spec 0018 に従う（`.` パスとは構文的に分離される）．

### Match 式

`match` は式であり，すべての arm は同じ型を返す．

pattern は上から順に判定される．

```emela
match value {
    Some(v) -> v
    None -> 0
}
```

enum に対する match は exhaustive である必要がある．

```emela
enum BoolLike {
    Yes
    No
}

let x = BoolLike::Yes

match x {
    Yes -> 1
    No -> 0
}

// reject
match x {
    Yes -> 1
}
```

pattern の variant は裸の `Variant`，または enum 名で修飾した `Enum::Variant` で書ける
（修飾も `::` を用い，`Enum.Variant` は使わない）．

unreachable pattern はエラーにする．

### Pattern guard

`match` 式の arm には pattern guard を付けることができる．

```emela
match value {
    Some(x) if x > 0 -> x
    Some(_) -> 0
    None -> -1
}
```

pattern が一致した後に評価される追加条件である．

```
pattern if condition -> expr
```

1. arm を上から順に調べる
2. pattern が一致するか調べる
3. pattern が一致した場合だけ guard を評価する
4. guard が true なら、その arm の右辺を評価する
5. guard が false なら、次の arm へ進む

guard の型は `Bool` でなければならない．

guard 内では、その pattern で束縛された変数を参照できる．

```emela
match user {
    User { age, name } if age >= 20 -> name
    User { name, ... } -> name
}
```

guard は式なので effect を持ちうる．

```emela
match path {
    Some(p) if exists(p) -> read(p)
    _ -> ""
}
```

この場合，exists(p) の effect は match 全体の effect に含まれる．

```
effects(match) =
    effects(scrutinee)
    ∪ effects(all guards)
    ∪ effects(all reachable arm bodies)
```

guard 付き arm は、同じ pattern の guard なし arm より前に置く必要がある。
```emela
match x {
    Some(_) -> 0
    Some(v) if v > 0 -> v
    None -> -1
}
```

実装はこれを error にする．

pattern guard 付き arm は，exhaustiveness check では完全な coverage とみなさない。

理由は guard が false になる可能性があるため．これは exhaustive ではない．

```emela
match x {
    Some(v) if v > 0 -> v
}
```

これは OK

```emela
match x {
    Some(v) if v > 0 -> v
    Some(_) -> 0
    None -> -1
}
```

### IR lowering

```emela
match value {
    Some(x) if x > 0 -> x
    Some(_) -> 0
    None -> -1
}
```

```
%v = eval value

switch_tag %v {
    Some => block_some
    None => block_none
}

block_some:
    %x = payload %v 0
    %g = gt.i32 %x 0
    branch %g block_some_guard_true block_some_guard_false

block_some_guard_true:
    return_match %x

block_some_guard_false:
    return_match 0

block_none:
    return_match -1

```
