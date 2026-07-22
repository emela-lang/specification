# Data Types

> **リファレンス** — 現在の規範。経緯・動機は末尾 [Provenance](#provenance) の RFC を参照。

ユーザー定義の直和型（`enum`）と積型（`record`）、それらの総称版、および
`match` による分解を定める。`Option` はこの機構で定義される Core Prelude の enum である。

キーワード MUST / MUST NOT / SHOULD / MAY は RFC 2119 に従う。

## 用語

| 用語 | 定義 |
|---|---|
| **enum** | 名前付き variant の直和型。 |
| **variant** | enum の選択肢。payload を持つものと持たないものがある。 |
| **record** | 名前付きフィールドを持つ積型。 |
| **再帰型** | 自身を payload / フィールド位置で参照する型。 |
| **lang-item** | コンパイラが役割で束縛する標準ライブラリの型（例 `Option`）。 |

## 1. enum

```emela
enum Color { Red  Green  Blue }
enum Either<L, R> { Left(L)  Right(R) }
```

- variant は enum 名で修飾した **型パス** `Enum::Variant` で参照・構築する。区切りは二重コロン `::` である（`.` ではない。`Color.Red` はエラー、→ [Modules の `::`](modules.md#3-名前解決順位)）。
- payload のない variant（`Color::Red`）はそれ自体が値である。payload を持つ variant（`Either::Left(1)`）は引数を伴って構築する。

## 2. match 式

`match` は式であり、すべての arm は同じ型を返す。pattern は上から順に判定される。

```emela
match value {
    Some(v) -> v
    None -> 0
}
```

- enum に対する match は **exhaustive** でなければならない (MUST)。到達しない pattern はエラーである。
- pattern の variant は裸の `Variant` または `Enum::Variant` で書ける（修飾も `::`）。
- **pattern guard**: `pattern if cond -> expr`。pattern が一致した後に `cond`（型は `Bool`、MUST）を評価し、`true` の arm のみ本体を評価する。guard 内では pattern で束縛した変数を参照できる。
- guard 付き arm は exhaustiveness で **完全な coverage とみなさない**（guard が false になりうるため）。また同じ pattern の guard なし arm より **前** に置かなければならない（後置は到達不能でエラー）。
- [effect](effects.md) は次の和集合である。

```text
effects(match) = effects(scrutinee) ∪ effects(all guards) ∪ effects(reachable arm bodies)
```

## 3. record

```emela
record User { id: Int  name: String }

let user = User { id: 1, name: "alice" }
user.name   -- "alice"
```

- record は名前付きフィールドを持つ。構築時に全フィールドを与え、フィールドは `r.field` でアクセスする。
- record は **組み込みの構造的等価を持たない**。`==` を使うには [`impl Eq`](traits.md#6-演算子の-trait-化) を与える。

## 4. ジェネリックデータ型

```emela
enum Either<L, R> { Left(L)  Right(R) }
record Pair<A, B> { first: A  second: B }
enum List<T> { Nil  Cons(T, List<T>) }
```

- 型パラメータは型名の直後の `<...>` に並べる（[Generics](generics.md) の関数と同じ構文規約、空リスト不可）。variant payload 型・フィールド型の中で型として使える。
- 型パラメータに **境界（`<T: Show>`）は付けられない** (MUST NOT)。データ型は能力を要求しない。能力は使用側の[関数の境界](traits.md#3-境界bounds)や[パラメータ付き impl](traits.md#2-impl-宣言)が要求する。
- 名前付き型は自身（および相互に）を payload / フィールド位置で参照してよい（**再帰型**、MUST）。payload / record は常にヒープ参照なので有限表現を持ち、構築順により非循環である。
- `Name<C1, …, Cn>` を型位置に書ける。型引数の数は宣言と一致しなければならない (MUST)。部分適用（`Either<Int>`）はできない。
- **構築と型引数の決定**: payload / フィールドの実引数型から[構造対応付け](generics.md#3-推論)で推論する。payload に現れない型パラメータや payload の無い variant（`Nil` 等）は **期待型** から決定する（[戻り値ディスパッチ](traits.md#7-戻り値ディスパッチと-monoid)と同じ規律）。期待型からも決まらなければエラー。
- ジェネリック enum への `match` は、scrutinee の具体型引数で payload の型が定まる。exhaustiveness は従来どおり variant 集合に対して検査する。

## 5. Option

`Option` は Core Prelude が提供するジェネリック enum であり、lang-item の役割 `option` に束縛される。

```emela
@lang("option")
enum Option<T> { Some(T)  None }
```

- 型検査・match・単相化・IR は §1–§4 の一般機構で扱う（`Option` 専用の組み込み型ではない）。
- `Some` / `None` は **修飾なしのベア名** で構築・照合できる（import 不要、従来の表面を保つ）。これは lang-item の特権である。
- **`?` は `Option` に対して定義しない** (MUST NOT、→ [Errors](errors.md#4--operator))。不在の伝播は `match` またはコンビネータ（`std.option`）で行い、error channel へ橋渡しするには `Option.ok_or` を用いる。
- `Option` は予約名であり、ユーザーソースが再定義することはできない (MUST NOT)。

（lang-item 機構は現状 [`specs/0041`](../specs/0041-lang-items.md)、Core Prelude 埋め込みは [`specs/0038`](../specs/0038-core-package-embedding.md)。）

## 未解決事項

- **variance**（現状は不変 invariant のみ）と、関数型フィールドの変性。
- 型の別名（type alias）、型パラメータの既定値・部分適用。
- 組み込み `Array<T>` を本機構上の宣言として再定義するか（`Option` は再定義済み、`Array` はプリミティブのまま）。
- `Result` を stdlib の値型として導入し `@lang("result")` で束縛するか（現状 error は [`throws`](errors.md) で表す）。

## Provenance

- enum・match・exhaustiveness・pattern guard: [`../specs/0005`](../specs/0005-enum-and-match-expression.md)
- record: [`../specs/0006`](../specs/0006-records.md)
- ジェネリックデータ型・再帰型: [`../specs/0028`](../specs/0028-generic-data-types.md)
- `Option` の Core-Prelude enum 化: [`../specs/0042`](../specs/0042-option-as-core-prelude-enum.md)（lang-item [`0041`](../specs/0041-lang-items.md) / core 埋め込み [`0038`](../specs/0038-core-package-embedding.md)）
