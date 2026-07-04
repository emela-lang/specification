## 0001: Core Values and Types

Status: Draft

### 概要

値，型，ヒープ値，参照，null の有無を定義する最重要仕様．

### 型の初期セットとその分類

```text
Unit
Bool
Int
Float
String
Array<T>
Record
Enum
Function
```

このうち primitive 型として評価されるのは `Unit`, `Bool`, `Int`, `Float`．

`String`, `Array<T>`, `Record`, `Enum payload`, `Closure environment` はすべてヒープに格納される．

### 仕様の方針

- primitive 値はコピー可能
- heap 値はランタイムが管理する参照として扱う．
- `String` は `StringRef` ヒープ参照
- `Array<T>` は `ArrayRef<T>` ヒープ参照
- `Record` はヒープ上の immutable な record として扱う
- `Enum` は tag + payload として扱う
- null は持たない
- nullable な値は `Option<T>` で表す
- 所有権，借用，move semantics は導入しない．
    - 基本的に GC に近い仕組みでメモリを管理する

### Examples

```emela
let name = "alice"
let xs = [1, 2, 3]
```

型は以下のとおりになる．

```text
name: String
xs: Array<Int>
```

IR での表現は次のような参照型に変化する．

```text
%name: StringRef
%xs: ArrayRef<Int>
```

Emela 自体は `StringRef` をサポートせず，ユーザーから見える型は `String` にする．

### Int / Float

`Int` は `i32`（32bit・2の補数・符号付き）に初期は固定する．

`Int` の算術 `+ - *` は **modulo 2^32 で wrap する** (MUST)．overflow は trap も panic もせず，
2の補数表現で一意に定まる値を生む．これは WASM の `i32.add` / `i32.sub` / `i32.mul` のネイティブな
意味と一致し，すべてのバックエンドで同一の結果を与える（0000: ターゲット間で観測可能な意味が変わら
ない）．例: `2147483647 + 1 == -2147483648`．

除算・剰余の意味（0方向切り捨て・0 除算 panic・`Int.min / -1` の扱い）は 0016 が定める．

checked / saturating な算術は言語組み込みには持たず，必要になれば stdlib 関数として導入する．

`Float` は `f64`（IEEE-754 binary64）に初期は固定する．演算は IEEE-754 に従う（`inf` / `nan` を含む，
0016）．

### Record

値的に等価比較できない．

### String

内部表現は UTF-8 の byte 列である．ただしユーザーから見える要素（文字）と length の単位は **Unicode scalar value**（`Char`, 0017）であり，byte ではない．添字・長さ・部分文字列はすべて scalar 単位で数える（0007 / 0017）．

### Never

`Never` は値を持たない型である（bottom type）．`Never` 型の値は存在しない．

通常評価が値を生成せずに終わる式の型を `Never` とする．`throw` 式と `panic` 式の型は `Never` であり (0011)，制御がそこから先へ進まないことを表す．

`Never` は bottom type であり，すべての型の部分型である（`Never <: T`）．したがって型推論において `Never` はいかなる制約も課さず，期待される型をそのまま採用する（`unify(Never, T) = T`）．`throw` や `panic` は，任意の型が期待される位置に置くことができる．

これは「まだ決まっていない型変数」とは異なる．`Never` は未確定なのではなく「あらゆる型に吸収される最小の型」であり，検査器は `Never` を全型の部分型として扱わなければならない (MUST)．

分岐式（`match` / `if`）の結果型は全 arm の型の最小上界 (join) であり，`Never` はその単位元である（`join(Never, T) = T`）．ある arm の型が `Never` でも，式全体の型はもう一方の arm の型になる．これが `throw` / `panic` を任意の arm に書ける根拠である．

```emela
fn lookup(id: UserId, table: Table) -> User throws NotFound uses {} {
    match Table.get(table, id) {
        Some(user) -> user
        None -> throw NotFound   -- throw の型 `Never` は期待型 `User` に適合する
    }
}
```

`Never` 型の値は存在しないため，`Never` を引数型やフィールド型に置いても，その値を構築・取得する手段はない．

`Never` は primitive でもヒープ値でもなく，値の分類には現れない．

`throws Never` は non-throwing（error を送出しない）と等価である (0011)．
