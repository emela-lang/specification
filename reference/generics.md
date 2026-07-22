# Generics

> **リファレンス** — 現在の規範。経緯・動機は末尾 [Provenance](#provenance) の RFC を参照。

関数と[データ型](data-types.md#4-ジェネリックデータ型)に型パラメータを持たせるパラメトリック多相。
型パラメータは **不透明** で、境界（trait 制約）は [Traits](traits.md#3-境界bounds) が、
effect-row の多相は [Effects](effects.md#7-effect-row-多相) が別に定める。

キーワード MUST / MUST NOT / SHOULD / MAY は RFC 2119 に従う。

## 用語

| 用語 | 定義 |
|---|---|
| **型パラメータ** | `<T, U>` で宣言する、具体型を後で受ける型変数。 |
| **不透明性（parametricity）** | 型パラメータの値に具体型固有の操作ができない性質。 |
| **単相化（monomorphization）** | 呼び出しごとに具体型へ特殊化してコードを生成すること。 |

## 1. 型パラメータの宣言

名前付き関数は、関数名の直後の `<...>` に型パラメータを列挙する。

```emela
fn identity<T>(x: T) -> T uses {} { x }
fn map<T, U>(xs: Array<T>, f: (T) -> U uses {}) -> Array<U> uses {} { ... }
```

- 型パラメータは1つ以上をカンマ区切りで並べる。空リスト `<>` は書けない。慣習として大文字始まり。
- 宣言した型パラメータは、そのシグネチャ（引数型・戻り値型・`throws` 型）と本体の中で **型** として使える。スコープはその関数定義に閉じ、異なる関数の同名パラメータは無関係である。

## 2. 不透明性（parametricity）

型パラメータ `T` の値に対してできるのは次に限る。

- `let` 束縛・引数として渡す・関数から返す
- `Array<T>` の要素にする / 取り出す、`Option<T>` 等に入れる / 取り出す
- `T` が関数型のパラメータ（`f: (A) -> T`）なら適用する

具体型を要求する操作——算術・比較・[`match`](data-types.md#2-match-式)・フィールドアクセス——は `T` に対してコンパイルエラーである。これらが必要なら[境界（trait）](traits.md#3-境界bounds)を付ける。この制約により、ジェネリック関数の観測可能な振る舞いは型引数によらず同一になる。

```emela
fn bad<T>(x: T) -> T uses {} { x + x }   -- error: `+` は具体的な数値型を要求する
```

## 3. 推論

- ジェネリック関数の呼び出しでは、型引数を **実引数の型から推論** する。明示的な型引数構文（`identity<Int>(x)`）は未導入。
- 推論は、宣言上のパラメータ型（型変数を含む）と実引数の具体型を構造的に対応付けて代入を求める。対応付けは `Array<D>` / `Option<D>` / `Name<D...>`（ユーザー定義ジェネリック型）/ 関数型 `(D...) -> Dr` を再帰的に貫く。
- すべての型パラメータが一意に定まらなければならない。そのため各型パラメータは **少なくとも1つのパラメータ型の中（ネストを含む）に出現していなければならない** (MUST)。どのパラメータ型にも現れない型パラメータはコンパイルエラーである。

```emela
fn first<T>(xs: Array<T>) -> Option<T> uses {} { array_get(xs, 0) }
first([1, 2, 3])       -- Array<T> ↔ Array<Int> → T = Int、結果は Option<Int>

fn pick<T>() -> T uses {}   -- error: `T` はどのパラメータ型にも現れない
```

## 4. effect と throws

- ジェネリック関数も具体的な [`uses`](effects.md) / [`throws`](errors.md) を宣言できる。
- effect row を型変数で受ける多相は [effect-row 多相](effects.md#7-effect-row-多相)（row 変数は sigil `'` 付きで `<...>` の外）。
- `throws E` の `E` に通常の型パラメータを用いること（単一の error 型を総称する）は有効である。複数 error 型の行多相（union error）は未導入。

## 5. 制限

- ジェネリックにできるのは **名前付き `fn` 定義** のみである。無名関数は型パラメータを宣言できない。
- 型パラメータに **境界は付かない**（完全に不透明）。境界（trait 制約）は [Traits](traits.md#3-境界bounds) が与える。
- ジェネリック関数は **直接呼び出し**（`name(args)`）でのみ使える。第一級の値として束縛・受け渡しすること（`let f = identity`）は、型引数が決まらないため許可しない。

## 6. 単相化と erasure

到達可能な `(関数, 具体型引数)` の組ごとに、型変数を具体型へ置換した単相版を生成する。特殊化は推移的に展開する。typed IR には型変数が一切現れず、backend は具体型だけを受け取る（[Traits の erasure](traits.md#8-解決の限界) と同じ不変条件）。[ジェネリックデータ型](data-types.md#4-ジェネリックデータ型)のレイアウトも同様に単相化される。

## 未解決事項

- **明示的な型引数構文**（`identity<Int>(x)` 相当、turbofish）と、比較演算子 `<` との曖昧性解決。
- どのパラメータ型にも現れない型パラメータ（`pick<T>() -> T`）を、明示型引数の導入とあわせて許可するか。
- **error 型に対する行多相**（union error）。
- ジェネリック関数を **第一級の値** として扱う手段。

## Provenance

- ジェネリック関数・不透明性・型引数推論・単相化: [`../specs/0014`](../specs/0014-generic-functions.md)
- ジェネリックデータ型: [`../specs/0028`](../specs/0028-generic-data-types.md)（→ [Data Types](data-types.md)）
- effect-row 多相: [`../specs/0022`](../specs/0022-effect-row-polymorphism.md)（→ [Effects](effects.md)）
- 型パラメータの境界: [`../specs/0020`](../specs/0020-traits.md)（→ [Traits](traits.md)）
