# 0027: Boolean and Comparison Operators

Status: Draft

論理演算子 `&& || !` と，導出比較演算子 `!= > >= <=` を定義する仕様．あわせて全演算子の優先順位表を
統合して規範化する．0015（If Expression）・0020（Traits, 演算子の trait 化）を拡張する．

## Summary

- 前置演算子 `!`（論理否定）と，短絡評価する二項演算子 `&&`（論理積）・`||`（論理和）を追加する．
  いずれも `Bool` 専用の**言語組み込み**であり，trait メソッドへは desugar しない．
- 比較 `!= > >= <=` を追加する．これらは新しい trait を導入せず，既存の `Eq.eq` / `Ord.lt`（0020）へ
  **導出 desugar** する．
- 全演算子の優先順位を一つの表として規範化する．

## Motivation

現状，言語には論理演算子が存在せず，複合条件（`a && b`，`n < 0 || n > 10`）が書けない．比較も `==` と
`<` しかない（0020 の演算子表）．`if`（0015）は `Bool` を要求するのに，`Bool` を合成する手段が無い．

`&&` / `||` は短絡評価（右辺を条件付きでのみ評価する）を要するため，値を両方受け取る trait メソッド
（0020）では表現できない．`if` と同類の**制御構文**として言語に置く．一方 `!= > >= <=` は `eq` / `lt`
から一意に導出できるため，新しい trait を増やさずに desugar で定義する．

## Specification

### 論理演算子（言語組み込み）

- **`!e`**: 前置演算子．`e` の型は `Bool` でなければならない (MUST)．結果は `Bool`．
- **`a && b`**: `if a { b } else { false }` と等価である (MUST)．
- **`a || b`**: `if a { true } else { b }` と等価である (MUST)．
- 両オペランドの型は `Bool` (MUST)．`&&` / `||` は**短絡評価**する: 左辺の値で結果が定まる場合，右辺は
  評価されない（上記の `if` 等価から従う）．
- effect（0009 / 0015）: `effects(a && b) = effects(a) ∪ effects(b)`（`if` と同じく，型検査では両辺を
  合算する）．`!` は `effects(e)` のまま．
- `! && ||` は trait メソッドへ desugar **しない** (MUST)．0020 の演算子 trait 化の対象外である
  （0021 の「言語規則はコンパイラが保持してよい」に属する）．

### 導出比較演算子（desugar）

次の等価が規範である (MUST)．新しい trait は導入しない．

| 式 | 等価な式 | 経由する trait (0020) |
| ------- | ----------- | ----------------- |
| `a != b` | `!(a == b)` | `Eq.eq` |
| `a > b`  | `b < a`     | `Ord.lt` |
| `a <= b` | `!(b < a)`  | `Ord.lt` |
| `a >= b` | `!(a < b)`  | `Ord.lt` |

- オペランド型に対する要求は `==` / `<` と同一である（`Eq` / `Ord` の instance が in-scope に必要，
  0020）．ユーザー型に `impl Ord` を与えれば `> <= >=` も使えるようになる．
- desugar は構文変換であり，オペランドの評価回数・評価順（左→右）を変えない (MUST)．
  `a > b` は `b < a` へ desugar されるが，**評価は `a`，`b` の順** である（脱糖は演算子の引数順のみを
  入れ替え，評価順序は元の式の字面に従う）．

### 優先順位（統合表）

強い順に，次を規範とする (MUST)．既存仕様（0016 / 0017 / 0019 / 0020）の定めと整合する．

```text
1. !                    (前置)
2. * / %                (左結合, 0016)
3. + - ++               (左結合, 0017)
4. == != < > <= >=      (左結合)
5. &&                   (左結合)
6. ||                   (左結合)
7. |>                   (左結合・最低, 0019)
```

- 比較（4）は算術（2, 3）より弱い: `a + 1 < b * 2` は `(a + 1) < (b * 2)`．
- 比較の連鎖 `a < b < c` は `(a < b) < c` と解釈され，`Bool` に `Ord` instance が無いため通常は型エラー
  になる（意図的：連鎖比較構文は提供しない）．
- `&&` は `||` より強い: `a || b && c` は `a || (b && c)`．

## Examples

```emela
fn in_range(n: Int, lo: Int, hi: Int) -> Bool {
    lo <= n && n <= hi
}

fn valid(user: User) -> Bool {
    user.age >= 20 && !is_banned(user) || is_admin(user)
    -- ((user.age >= 20) && (!is_banned(user))) || is_admin(user)
}
```

短絡評価:

```emela
fn safe_head(xs: Array<Int>) -> Int {
    if length(xs) > 0 && get_or_panic(xs, 0) > 0 { 1 } else { 0 }
    -- length(xs) == 0 のとき get_or_panic は評価されない
}
```

## Compilation Notes

この節は非規範的である．

- `&&` / `||` は `if` 等価式へ frontend で desugar してよい（0015 の分岐 lowering を再利用）．
  WebAssembly では `(if (result i32) ...)`, JavaScript ではネイティブの `&&` / `||` に直接 lower
  してもよい（意味が一致するため）．
- `!` は `i32.eqz` / JS `!`．
- 導出比較は desugar 後に 0020 の trait 解決（組み込み型は intrinsic へインライン, 0021）に乗るため，
  backend の追加対応は不要．

## Open Questions

- `Bool` に対する `Eq`（`==`）instance を Core Prelude に追加するか（現状 0020 の instance は
  `Int` / `Float` のみ．`Bool` の比較は `if` で書ける）．
- ビット演算（`& | ^ << >>`）の導入時期と，`&` / `&&` の字句的関係（0019 の `|>` と `|` の棲み分けと
  共有）．
