# 0022: Effect-Row Polymorphism

Status: Draft

関数の `uses` 節（effect row, 0009）を **effect-row 変数** で受け取り，伝播できるようにする仕様．
`map` / `filter` / `fold` のような高階関数を，純粋な関数にも effectful な関数にも **同一定義** で
適用できるようにする．spec 0014（Generic Functions）を effect row に拡張する．

## Summary

- effect-row 変数は，先頭に sigil `'` を付けた識別子（`'e`）で書く．**型パラメータ `<...>` とは別カテゴリ**
  であり，`<...>` には宣言しない．
- `'e` は `uses` の位置にのみ現れる．具体 effect（`io`, `fs` 等，0009）や型と字句的に区別され，曖昧に
  ならない（`uses { io }` は具体，`uses 'e` は変数）．
- effect-row 変数は，その関数のシグネチャに **暗黙に全称量化** される．宣言は不要で，`uses` 位置に現れた
  時点で束縛される．スコープはその関数定義に閉じ，異なる関数の同名 `'e` は無関係である（0014 の型
  パラメータと同じ規律）．
- 呼び出し時，effect-row 変数は **実引数（関数値）の effect row から推論** する（0014 の型引数推論の
  effect 版）．
- effect row は静的な契約であり実行時表現を持たないため，effect-row 多相は **単相化を必要としない**
  （0014 の型単相化と異なり，検査後に消去するだけ）．

## Motivation

0009 の effect row と 0014 のジェネリクスはあるが，0014 は effect row を型変数にできず，高階関数の
`uses` を具体的に固定するしかなかった．そのため純粋関数用の `map`（`uses {}`）と effectful 用の
`map`（`uses { io }` など）を別々に定義する必要があり，関数型言語の中核である高階関数が effect を
跨げなかった．

また，effect row は「型」ではなく effect の集合（row）であり，実行時表現を持たず単相化もされない点で
型パラメータと根本的に異なる．この2つを同じ `<...>` に並べると，「単相化される型」と「erase される row」，
さらに小文字の具体 effect（`io`）が同居して混乱する．本仕様は effect-row 変数を sigil `'` 付きの
**別カテゴリ** として `<...>` の外に置き，この混乱を避ける．

複数の仕様がこの機能を前方参照している．

- 0003 は `fn apply<T>(x, f: (T) -> T uses 'e) -> T uses 'e` を予告する．
- 0008 は `fn map<T, U>(xs, f: (T) -> U uses 'e) -> Array<U> uses 'e` を予告する．
- 0014 は effect-row 多相を「本仕様では扱わない」とし，本仕様へ送った．

## Specification

本仕様は 0014 を拡張する．0014 の型引数推論・「各パラメータは少なくとも 1 つのパラメータ位置に
出現しなければならない」規則を，effect-row 変数へ一般化する．

### effect-row 変数の記法

effect-row 変数は，先頭に sigil `'` を付けた識別子で書く（`'e`, `'e1` 等）．

```emela
fn map<T, U>(xs: Array<T>, f: (T) -> U uses 'e) -> Array<U> uses 'e {
    ...
}
```

- **`<...>` は型パラメータ専用である** (MUST)．effect-row 変数を `<...>` に書いてはならない (MUST NOT)．
- `'e` は **`uses` の位置にのみ** 現れる (MUST)．型の位置（引数型・戻り値型・`throws` 型）には書けない．
- `'e` は宣言不要で，`uses` 位置に現れた時点でそのシグネチャに暗黙に全称量化される．同一シグネチャ内の
  同名 `'e` は同一の row 変数を指す．スコープはその関数定義に閉じる．
- sigil `'` により，effect-row 変数は具体 effect（`io` 等）とも型とも字句的に区別される．

### 使用位置

effect-row 変数 `'e` は次に書ける．

- パラメータの関数型の `uses` 節: `f: (T) -> U uses 'e`
- 当該関数自身の `uses` 節: `-> Array<U> uses 'e`

第一段階では，`uses` 節は次のいずれかに限る (MUST)．

- 具体的な effect row（`uses {}`, `uses { io }` 等，0009）
- 単一の effect-row 変数（`uses 'e`）

具体 effect と row 変数の混在（row 拡張，例 `uses { io, ..'e }`）は spec 0023 (Effect Inference and Subsumption) が本制限を緩和して定義する．

### 推論

effect-row 変数は，実引数の関数値の effect row から推論する．0014 の型引数推論と同じ構造対応付けを
`uses` 位置に対して行う．

```emela
fn map<T, U>(xs: Array<T>, f: (T) -> U uses 'e) -> Array<U> uses 'e

map([1, 2, 3], fn (x: Int) -> Int uses {} { x + 1 })
-- f の effect row は {} なので 'e = {}．map 全体は uses {}

map(paths, read)   -- read: (Path) -> String uses { fs }
-- f の effect row は { fs } なので 'e = { fs }．map 全体は uses { fs }
```

- 各 effect-row 変数は，少なくとも 1 つのパラメータの `uses` 位置に現れなければならない (MUST)（0014 の
  「型パラメータは少なくとも 1 つのパラメータ型に現れる」の effect 版）．いずれの `uses` 位置にも現れない
  row 変数は推論できないためコンパイルエラーとする．
- 呼び出し式全体の effect row は，推論した代入を関数自身の `uses` 節へ適用したものになる．

### 伝播と検査

効果検査（0009）は，具体化後の effect row に対して行う．effect-row 多相な関数を呼ぶと，その呼び出し式の
effect は代入後の row になり，呼び出し元へ通常どおり伝播する．純粋（`uses {}`）な文脈から，`'e = {}` に
具体化された呼び出しを行うことは許される（純粋関数を渡した `map` は純粋）．

### throws との関係

`throws E` は単一の error **型** を宣言する（0011）．したがって `E` に通常の型パラメータを用いる

```emela
fn map_throws<T, U, E>(xs: Array<T>, f: (T) -> U throws E) -> Array<U> throws E
```

は **0014 の通常の型パラメータ** であり，`E` は型なので `<...>` に置く．これは本仕様の対象ではない
（既に 0014 で有効）．

複数の error 型を開いた row として束ねる **error-row 多相**（`throws { A, ..'r }` 的な行変数）は，型では
なく row であるため，導入する場合は effect-row 変数と同様に sigil `'` 付きで `<...>` の外に置く．本仕様
では扱わない（0011 / 0014 の Open Questions）．

## Examples

代表的な高階関数は，同一定義で pure / effectful の両方に使える．

```emela
fn map<T, U>(xs: Array<T>, f: (T) -> U uses 'e) -> Array<U> uses 'e { ... }
fn filter<T>(xs: Array<T>, pred: (T) -> Bool uses 'e) -> Array<T> uses 'e { ... }
fn fold<T, A>(xs: Array<T>, init: A, f: (A, T) -> A uses 'e) -> A uses 'e { ... }
```

```emela
map([1, 2, 3], fn (x: Int) -> Int uses {} { x + 1 })            -- uses {}
map(paths, fn (p: Path) -> String uses { fs } { read(p) })      -- uses { fs }
```

## Compilation Notes

この節は非規範的である．

- **単相化不要**: effect row は静的な契約で実行時表現を持たない（0009 / 0012）．したがって effect-row
  変数は型検査後に消去するだけでよく，0014 の型単相化のような特殊化を要しない．型パラメータ `T` / `U` の
  単相化は従来どおり行い，effect row は erase する．同じ `map<T, U>` でも，`'e` の違いだけで別の特殊化を
  生む必要はない（`T` / `U` が同じなら単一の特殊化を共有する）．
- typed IR（0012）は effect 注釈を保つが，具体化後の row のみが現れ，row 変数は残らない．

## Open Questions

- ~~row 拡張（`uses { io, ..'e }`）~~ → spec 0023 で定義済み．
- 複数 row 変数間の順序・重複の正規化と，row の等価判定．
- **error-row 多相**（`throws { A, ..'r }` 的な行変数 / union error）の導入（0011 / 0014 の Open Questions
  と連動）．導入時は sigil `'` 付きで `<...>` の外に置き，本仕様の effect-row 変数と同じ扱いとする．
- effect-row 変数の明示指定構文（0014 の明示型引数と連動）．
- effect の subsumption は spec 0023 が具体 row 同士について定義した．row 変数を通した緩和
  （bounded row variable）は未解決（0023 の Open Questions と共有）．
- 無名関数（lambda）が effect-row 多相を持てるようにするか（0014 は無名関数の型パラメータを禁止）．
