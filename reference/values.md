# Values & Types

> **リファレンス** — 現在の規範。経緯・動機は末尾 [Provenance](#provenance) の RFC を参照。

値・型・その実行時分類、およびプリミティブ演算の観測可能な意味を定める。
null は持たず、値の不在は [`Option`](data-types.md#5-option) で表す。所有権・借用・move は導入しない。

キーワード MUST / MUST NOT / SHOULD / MAY は RFC 2119 に従う。

## 用語

| 用語 | 定義 |
|---|---|
| **primitive 値** | コピー可能な値（`Unit` / `Bool` / `Int` / `Float` / `Char`）。 |
| **heap 値** | ランタイムが管理する参照として扱う値（`String` / `Array` / `Record` / `Enum` payload / closure 環境）。 |
| **`Never`** | 値を持たない型（bottom type）。 |

## 1. 型の分類

| 型 | 分類 | 実行時表現 |
|---|---|---|
| `Unit` `Bool` `Int` `Float` `Char` | primitive | 値（コピー可能） |
| `String` `Array<T>` `Record` `Enum` payload・closure 環境 | heap | ランタイム管理の参照 |

- null は持たない。nullable な値は [`Option<T>`](data-types.md#5-option) で表す。
- 所有権・借用・move semantics は導入しない。メモリはランタイムが管理する非循環ヒープである（→ Memory Model、現状 [`specs/0024`](../specs/0024-memory-model.md) / [`specs/0048`](../specs/0048-arc-wasm.md)）。

## 2. Int

- `Int` は `i32`（32bit・2の補数・符号付き）に固定する。
- 算術 `+ - *` は **modulo 2³² で wrap する** (MUST)。overflow は trap も panic もしない。例：`2147483647 + 1 == -2147483648`。WASM の `i32.add` 等と一致し、全 backend で同一結果になる。
- 除算 `/`：**0方向への切り捨て**（truncated）。例：`7 / 2 == 3`、`-7 / 2 == -3`。
- 剰余 `%`：被除数の符号を持ち、`a == (a / b) * b + (a % b)` を満たす。例：`7 % 2 == 1`、`-7 % 2 == -1`。
- **0 による整数の除算・剰余は panic する** (MUST、[回復不能](errors.md#6-panic))。lower 先は trap。
- **`Int.min / -1`（`-2147483648 / -1`）は panic する** (MUST)。結果が i32 で表現できないため。加算等の wrap とは異なり除算は wrap しない。
- **`Int.min % -1` は `0` である** (MUST)。
- checked / saturating な算術は言語組み込みには持たず、必要なら stdlib 関数として導入する。

## 3. Float

- `Float` は `f64`（IEEE-754 binary64）に固定する。演算は IEEE-754 に従う（`inf` / `nan` を含む）。
- `/` は実数除算。`Float / 0.0` は IEEE-754 に従い（`inf` / `nan`）**trap しない**。
- `%` は `Float` に対しては型エラーである（`%` は `Int % Int -> Int` のみ）。

## 4. Bool・Unit

- `Bool` は `true` / `false`。論理演算子と `if` の条件は `Bool` を要求する（→ [Expressions](expressions.md)）。
- `Unit` は値を1つだけ持つ型。block の最終式が束縛のとき等、値を持たない位置の型になる（→ [Expressions](expressions.md#1-block)）。

## 5. Char・String

- **`Char`** は1つの Unicode scalar value を表す primitive 型（コピー可能）。文字リテラルは `'x'`（エスケープは文字列と同じ `\n \t \\ \' \"`）。
- **`String`** は immutable な text である。内部表現は UTF-8 の byte 列だが、**ユーザーから見える要素と length の単位は Unicode scalar value（`Char`）** であり、byte ではない。添字・長さ・部分文字列はすべて scalar 単位で数える。
- 純粋操作（すべて `uses {}`、`throws` 無し）：
  - `a ++ b`：`String ++ String -> String`（連結、結合的・左結合、新しい `String` を生成し両辺は不変）。
  - `char_from_code(n: Int) -> Char`：コードポイント `n` の `Char`。
  - `string_from_char(c: Char) -> String`：1文字の `String`。

## 6. Array

- `Array<T>` は固定長の配列である（可変長配列は別途）。要素は同一型 `T`。値は不変で、`array_push` は末尾に足した **新しい配列** を返す（元は変更しない）。
- 基本操作はベア名の [intrinsic](runtime-boundary.md#4-intrinsic純粋演算)（Core Prelude が宣言、backend が native 命令へ lower）と、その安全ラッパで供給する。

```emela
intrinsic fn array_length<T>(a: Array<T>) -> Int uses {}
intrinsic fn array_get_unchecked<T>(a: Array<T>, i: Int) -> T uses {}   -- 0 <= i < length 前提
intrinsic fn array_push<T>(a: Array<T>, x: T) -> Array<T> uses {}
```

- 安全な添字アクセスは `pub fn array_get<T>(a: Array<T>, i: Int) -> Option<T>`。境界外は `None` を返し panic しない（[不在は `Option`](errors.md#1-失敗の3分類)）。

## 7. Bytes

- `Bytes` は **解釈を持たない生バイトの列**（immutable）である。要素は byte（`Int` で 0–255）、長さ・添字はすべて **byte 単位** で数える（`String` の scalar 単位と対比）。実行時表現は `String` と同一（`[len][bytes]`）で、違いは「byte 単位・無解釈か（Bytes）／scalar 単位・UTF-8 か（String）」だけである。
- 基本操作はベア名 intrinsic ＋ `std.bytes` の安全ラッパで供給する。
  - `bytes_length` / `bytes_get_unchecked` / `bytes_slice`（半開区間 `[start, end)`、範囲はクランプ）/ `bytes_concat`。
  - `byte_at(b, i) -> Option<Int>`：境界外は `None`。
- `String` ⇄ `Bytes`：`to_bytes(s: String) -> Bytes`（UTF-8 encode、全域）／`string_from_bytes(b: Bytes) -> Option<String>`（UTF-8 decode、不正なら `None`。panic しない）。
- 演算子 `++`（`impl Concat for Bytes`）・`==`（`impl Eq for Bytes`、byte 列の逐次比較）が使える（→ [Traits](traits.md#6-演算子の-trait-化)）。
- 専用リテラル（`b"..."`）は未導入。空の `Bytes` は `to_bytes("")` で得る。

## 8. Never

`Never` は値を持たない bottom type である。`Never` 型の値は存在しない。

- 通常評価が値を生成せず終わる式の型を `Never` とする。[`throw`](errors.md#2-throws-と-throw) 式と [`panic`](errors.md#6-panic) 式の型は `Never` である。
- `Never` はすべての型の部分型である（`Never <: T`）。型推論において `unify(Never, T) = T`、分岐式の join で `join(Never, T) = T`。ゆえに `throw` / `panic` を任意の arm・任意の期待型の位置に置ける。
- `Never` は primitive でもヒープ値でもなく、値の分類には現れない。`throws Never` は [非送出](errors.md#7-エントリポイント)と等価である。

## 演算子の意味論について

`/` `%` `++` `==` などの演算子は[対応する trait メソッドへ desugar](traits.md#6-演算子の-trait-化) される。上記の数値・文字・文字列の **観測可能な意味**（切り捨て・0除算 trap・wrap・連結）は、その組み込み instance を backing する [intrinsic の lowering](runtime-boundary.md#4-intrinsic純粋演算) によって保存される (MUST)。

## 未解決事項

- **可変配列 / 可変長配列**（現状 `Array<T>` は固定長・不変）と効率的な builder。
- **`String` / `Bool` の `Eq`**（`==`）を Core Prelude に入れるか（現状の組み込み `Eq` instance は `Int` / `Float`。`Bytes` は本章で提供）。
- 無効コードポイント（サロゲート等）に対する `char_from_code` の挙動（trap / クランプ / 下位ビット）。
- `Char` の比較・`Char -> Int`（`Char.code`）など他の `Char` 演算。
- **byte string literal**（`b"..."`）と `Bytes` の hex 表現（`Show`）。
- `Int` の幅（i32 固定を超える整数型）、`Int` 自体を stdlib 型にする拡張余地。

## Provenance

- 値・型の分類・`Int`/`Float` 表現・`Never`: [`../specs/0001`](../specs/0001-core-values-and-types.md)
- `String` / `Array`: [`../specs/0007`](../specs/0007-array-and-strings.md)
- 整数の除算・剰余: [`../specs/0016`](../specs/0016-integer-division-and-remainder.md)
- `Char`・文字列連結・変換: [`../specs/0017`](../specs/0017-char-and-string-concatenation.md)
- `Bytes`: [`../specs/0051`](../specs/0051-bytes.md)
- 演算子を backing する intrinsic: [`../specs/0021`](../specs/0021-intrinsics.md)（→ [Runtime Boundary](runtime-boundary.md)）
