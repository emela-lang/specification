# Expressions & Control Flow

> **リファレンス** — 現在の規範。経緯・動機は末尾 [Provenance](#provenance) の RFC を参照。

Emela は式指向である。block・`if`・[`match`](data-types.md#2-match-式) はすべて **式** であり、
文と式の区別も明示的な `return` も持たない。

キーワード MUST / MUST NOT / SHOULD / MAY は RFC 2119 に従う。

## 1. block

block は式である。

```emela
{
    let x = 1
    let y = 2
    x + y        -- block の値
}
```

- block の型・値は **最後の式** による。最後の式が束縛のときは `Unit` に fallback する。
- 関数本体も block 式である。
- 明示的な `return` 構文は用意しない。

## 2. `if`

```text
if cond { then_block } else { else_block }
```

- `cond` の型は `Bool` でなければならない (MUST)。
- `then_block` と `else_block` は [block 式](#1-block)であり、その型は一致しなければならない (MUST)。この共通型 `T` が `if` 式全体の型になる。
- **`else` 節は必須** である (MUST)。
- 評価は `cond` を評価し、`true` なら `then_block` のみ、`false` なら `else_block` のみを評価する（選ばれない branch は評価しない）。
- [effect](effects.md) は `cond`・`then_block`・`else_block` の和集合である（実行時に branch が決まるため、型検査では両 branch を合算する）。
- 文的に使いたい場合は両 branch を `Unit` にする。

## 3. `match`

`match` は式である。exhaustiveness・pattern guard を含む詳細は [Data Types の match](data-types.md#2-match-式) が定める。

## 4. 論理演算子

`! && ||` は `Bool` 専用の **言語組み込み** であり、trait メソッドへ desugar しない (MUST)。

- `!e`：前置。`e` は `Bool` (MUST)、結果は `Bool`。
- `a && b` ≡ `if a { b } else { false }` (MUST)。
- `a || b` ≡ `if a { true } else { b }` (MUST)。
- 両オペランドは `Bool` (MUST)。`&&` / `||` は **短絡評価** する（上記 `if` 等価から従う）。
- effect は `effects(a) ∪ effects(b)`（`!` は `effects(e)` のまま）。

## 5. 導出比較演算子

次の等価が規範である (MUST)。新しい trait は導入せず、既存の [`Eq.eq` / `Ord.lt`](traits.md#6-演算子の-trait-化) へ desugar する。

| 式 | 等価な式 | 経由する trait |
|---|---|---|
| `a != b` | `!(a == b)` | `Eq.eq` |
| `a > b` | `b < a` | `Ord.lt` |
| `a <= b` | `!(b < a)` | `Ord.lt` |
| `a >= b` | `!(a < b)` | `Ord.lt` |

- オペランド型への要求は `==` / `<` と同一である（`Eq` / `Ord` の instance が in-scope に必要）。ユーザー型に `impl Ord` を与えれば `> <= >=` も使えるようになる。
- desugar は構文変換であり、評価回数・評価順（左→右）を変えない (MUST)。`a > b` は `b < a` へ desugar されるが、評価は `a`、`b` の順である。

## 6. pipeline `|>`

左辺の値を右辺の呼び出しの第一引数へ流し込む純粋な構文糖である。型・effect・throws・実行時の意味はすべて脱糖後の呼び出しと一致する（新しい意味論は導入しない）。

- **第一引数挿入**: 右辺が呼び出し `f(a1, …, an)`（n ≥ 0）のとき、`lhs |> f(a1, …, an)` ≡ `f(lhs, a1, …, an)` (MUST)。
- **bare 右辺**: 右辺が呼び出しでない式 `e` のとき、`lhs |> e` ≡ `e(lhs)` (MUST)。`e` は関数値に評価されなければならない。
- **最低優先順位・左結合**: `a |> f |> g` ≡ `g(f(a))`。`a + b |> f` ≡ `f(a + b)`。
- **末尾 `?`**: 右辺末尾の [error 伝播 `?`](errors.md#4--operator) は挿入の **後** に適用する。`lhs |> g?` ≡ `(lhs |> g)?`。
- **評価順**: 脱糖後の呼び出しに従う。左辺は常に最初に、ちょうど一度だけ評価される。

```emela
xs |> map(double) |> filter(is_even) |> fold(0, add)
-- fold(filter(map(xs, double), is_even), 0, add)
```

## 7. 優先順位（統合表）

強い順（すべて左結合）：

```text
1. !                    (前置)
2. * / %
3. + - ++
4. == != < > <= >=
5. &&
6. ||
7. |>                   (最低)
```

- 比較（4）は算術（2, 3）より弱い：`a + 1 < b * 2` は `(a + 1) < (b * 2)`。
- `&&` は `||` より強い：`a || b && c` は `a || (b && c)`。
- 連鎖比較 `a < b < c` は `(a < b) < c` と解釈され、`Bool` に `Ord` instance が無いため通常は型エラーになる（連鎖比較構文は提供しない）。

## 未解決事項

- **`else` の無い `if`**（`Unit` 用）・**`else if`** 糖衣の明示化（現状は `else { if ... }`）。
- pipeline のプレースホルダ構文（`x |> f(a, _)`）と修飾パス右辺（`x |> int.to_string`）。
- `&&` / `|>` と将来のビット演算子（`& |`）の字句的棲み分け。

## Provenance

- block・式指向・`return` 非導入: [`../specs/0004`](../specs/0004-blocks-and-expression-semantics.md)
- `if` 式: [`../specs/0015`](../specs/0015-if-expression.md)
- 論理演算子・導出比較・優先順位表: [`../specs/0027`](../specs/0027-boolean-and-comparison-operators.md)
- pipeline `|>`: [`../specs/0019`](../specs/0019-pipeline-operator.md)
- `match` 式: [`../specs/0005`](../specs/0005-enum-and-match-expression.md)（→ [Data Types](data-types.md)）
