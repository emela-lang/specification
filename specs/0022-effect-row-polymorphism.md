# 0022: Effect-Row Polymorphism

Status: Draft

関数の `uses` 節（effect row, 0009）を **effect-row 変数** で受け取り，伝播できるようにする仕様．
`map` / `filter` / `fold` のような高階関数を，純粋な関数にも effectful な関数にも **同一定義** で
適用できるようにする．spec 0014（Generic Functions）を effect row に拡張する．

2026-07-23 改訂: row 変数を「sigil `'` 付き・`<...>` の外・暗黙全称量化」（`uses 'e`）から，
**sigil なしの小文字識別子を型パラメータリスト `<...>` に明示宣言する方式**（`fn map<T, U, e>`）へ
変更した（Motivation 参照）．

## Summary

- effect-row 変数（**row パラメータ**）は，sigil のない **小文字始まりの識別子**（`e`, `e1` 等）で
  書き，型パラメータと同じ `<...>` に **明示宣言** する (MUST): `fn map<T, U, e>(...)`.
- `<...>` 内は先頭文字で振り分ける: **大文字始まりは型パラメータ（0014），小文字始まりは row
  パラメータ**．型パラメータは大文字で始めなければならない (MUST, 0014 を改訂)．
- row 変数は `uses` の位置にのみ現れる (MUST)．bare 形 `uses e`（`uses { ..e }` と等価）と，
  拡張形 `uses { Io, ..e }`（0023 の row 拡張）の両方を書ける．具体 effect 名は大文字始まり
  （0037）なので，row 変数とは字句的に区別される．
- `<...>` で宣言されていない row 変数の使用はエラーである (MUST)．スコープはその関数定義に閉じる．
- 呼び出し時，row 変数は **実引数（関数値）の effect row から推論** する（0014 の型引数推論の
  effect 版）．
- effect row は静的な契約であり実行時表現を持たないため，effect-row 多相は **単相化を必要としない**
  （0014 の型単相化と異なり，検査後に消去するだけ）．

## Motivation

0009 の effect row と 0014 のジェネリクスはあるが，0014 は effect row を型変数にできず，高階関数の
`uses` を具体的に固定するしかなかった．そのため純粋関数用の `map`（`uses {}`）と effectful 用の
`map`（`uses { Io }` など）を別々に定義する必要があり，関数型言語の中核である高階関数が effect を
跨げなかった．

旧版は「単相化される型と erase される row を同じ `<...>` に並べると混乱する」ことを理由に，row 変数を
sigil `'` 付きの別カテゴリとして `<...>` の外に置き，`uses` 位置への出現で暗黙に全称量化していた．
本改訂はこれを反転し，`<...>` への明示宣言に統一する．理由:

- **束縛の明示性**．暗黙量化では「この `'e` はどこで束縛されたのか」を読み手がシグネチャ全体から
  探す必要があった．`<T, U, e>` は型パラメータと同じく量化点が一目で分かり，pub 境界（0000 原則5）
  の検査も「`uses` 位置の row 変数は `<...>` に宣言済みか」という単純な規則になる．
- **字句の単純さ**．新しい sigil トークンを増やさない．row 変数は普通の識別子であり，具体 effect 名
  （大文字始まり，0037）との区別は case 規約が与える．
- **混同の懸念は他の手段で解消される**．「単相化される型と erase される row の同居」は case 規約で
  読み分けられ，mangling / 特殊化に row が関与しないことは Compilation Notes が定める．`throws` の
  error 型パラメータ `E` が既に `<...>` に住んでいる（0014）こととも整合的である．

複数の仕様がこの機能を前方参照している．

- 0003 は `fn apply<T, e>(x, f: (T) -> T uses e) -> T uses e` を予告する．
- 0008 は `fn map<T, U, e>(xs, f: (T) -> U uses e) -> Array<U> uses e` を予告する．
- 0014 は effect-row 多相を「本仕様では扱わない」とし，本仕様へ送った．

## Specification

本仕様は 0014 を拡張する．0014 の型引数推論・「各パラメータは少なくとも 1 つのパラメータ位置に
出現しなければならない」規則を，effect-row 変数へ一般化する．

### row パラメータの宣言と記法

```emela
fn map<T, U, e>(xs: Array<T>, f: (T) -> U uses e) -> Array<U> uses e {
    ...
}
```

- row パラメータは，関数名直後の `<...>` に **小文字始まりの識別子** で宣言する (MUST)．型パラメータ
  （大文字始まり）と同じリストに並べ，順序は自由である．`<...>` 内の名前は型パラメータ・row
  パラメータを通して重複してはならない (MUST NOT)．
- 型パラメータは **大文字で始めなければならない** (MUST)（0014 の「慣習として大文字」を規範へ格上げ）．
- row パラメータを宣言できるのは **名前付き `fn` 定義** のみである．enum / record / trait の
  型パラメータリストに row パラメータは書けない (MUST NOT)．
- row パラメータに trait 境界（0020）は付けられない (MUST NOT)．
- row 変数は **`uses` の位置にのみ** 現れる (MUST)．型の位置（引数型・戻り値型・`throws` 型）には
  書けない．
- `uses` 位置に現れる row 変数は，その関数の `<...>` で宣言済みでなければならない (MUST)．未宣言は
  コンパイルエラーである．スコープはその関数定義（本体内の無名関数を含む）に閉じ，異なる関数の
  同名 row パラメータは無関係である（0014 の型パラメータと同じ規律）．

### `uses` 節の形

row 変数を含む `uses` 節は次のいずれかで書く．

- **bare 形**: `uses e` — 単一の row 変数．`uses { ..e }` と等価である．
- **拡張形**: `uses { Io, ..e }` — 具体 effect と row 変数 tail の和（0023 の row 拡張）．

制限:

- **パラメータの関数型の `uses` に書ける tail は最大 1 個である** (MUST)．複数 tail への分配は
  推論の解が一意に定まらないため許さない．
- **当該関数自身の `uses` には複数の tail を書いてよい** (MAY)．`uses { ..e1, ..e2 }` は
  `e1 ∪ e2` を意味する．

### 使用位置

row 変数 `e` は次に書ける．

- パラメータの（直接の）関数型の `uses` 節: `f: (T) -> U uses e`
- 当該関数自身の `uses` 節: `-> Array<U> uses e` / `uses { Log, ..e }`
- 本体内の無名関数の `uses` 節（囲む関数の row パラメータの参照）(MAY)

次には書けない（v1 制限, MUST NOT）．

- 戻り値型の中の関数型の `uses`
- パラメータの関数型のさらに内側（引数位置・戻り値位置）の関数型の `uses`

### 推論

row 変数は，実引数の関数値の effect row から推論する．0014 の型引数推論と同じ構造対応付けを
`uses` 位置に対して行う．

```emela
fn map<T, U, e>(xs: Array<T>, f: (T) -> U uses e) -> Array<U> uses e

map([1, 2, 3], fn (x: Int) -> Int uses {} { x + 1 })
-- f の effect row は {} なので e = {}．map 全体は uses {}

map(paths, read)   -- read: (Path) -> String uses { Fs }
-- f の effect row は { Fs } なので e = { Fs }．map 全体は uses { Fs }
```

- 各 row パラメータは，少なくとも 1 つのパラメータの（直接の）関数型の `uses` 位置に現れなければ
  ならない (MUST)（0014 の「型パラメータは少なくとも 1 つのパラメータ型に現れる」の effect 版）．
  いずれの `uses` 位置にも現れない row パラメータは推論できないためコンパイルエラーとする．
- 単一化は **最小解** を取る（0023）: 実引数の row `(A, tail Ta)` を宣言 `{ D, ..e }` に対応付ける
  とき，`e = (A \ D) ∪ Ta` とする (MUST)．
- 同一 row 変数が複数のパラメータの `uses` に現れる場合，各実引数からの残差の **和**（最小上界）に
  束縛する (MUST)．row は集合なので，型変数の「矛盾する束縛はエラー」（0014）と異なり和が定義できる．
- 呼び出し式全体の effect row は，推論した代入を関数自身の `uses` 節へ適用したものになる．

### 伝播と検査

効果検査（0009 / 0023）は，具体化後の effect row に対して行う．effect-row 多相な関数を呼ぶと，
その呼び出し式の effect は代入後の row になり，呼び出し元へ通常どおり伝播する．純粋（`uses {}`）な
文脈から，`e = {}` に具体化された呼び出しを行うことは許される（純粋関数を渡した `map` は純粋）．

呼び出し元自身が row 多相であれば，代入後の row に呼び出し元の row 変数が tail として現れてよい．
本体の推論 row ⊆ 宣言 row の上界検査（0023）は，tail を含む包含（0023 の成分ごと包含）で行う．

effect 操作の直接呼び出し（`Io.print` 等）に対する `uses` ゲート（0037）は，宣言 row の **具体部**
に対してのみ働く．tail は `e = {}` に具体化されうるため，操作呼び出しの免許にはならない．

### throws との関係

`throws E` は単一の error **型** を宣言する（0011）．したがって `E` に通常の型パラメータを用いる

```emela
fn map_throws<T, U, E>(xs: Array<T>, f: (T) -> U throws E) -> Array<U> throws E
```

は **0014 の通常の型パラメータ** であり，本仕様の対象ではない（既に 0014 で有効）．`E` は型
（大文字始まり），`e` は row（小文字始まり）として同じ `<...>` に同居する:
`fn f<T, E, e>(...)`.

複数の error 型を開いた row として束ねる **error-row 多相**（`throws { A, ..r }` 的な行変数）は
本仕様では扱わない（0011 / 0014 の Open Questions）．導入する場合は row パラメータと同じく
`<...>` 内の宣言とする．

## Examples

代表的な高階関数は，同一定義で pure / effectful の両方に使える．

```emela
fn map<T, U, e>(xs: Array<T>, f: (T) -> U uses e) -> Array<U> uses e { ... }
fn filter<T, e>(xs: Array<T>, pred: (T) -> Bool uses e) -> Array<T> uses e { ... }
fn fold<T, A, e>(xs: Array<T>, init: A, f: (A, T) -> A uses e) -> A uses e { ... }
```

```emela
map([1, 2, 3], fn (x: Int) -> Int uses {} { x + 1 })              -- uses {}
map(paths, fn (p: Path) -> String uses { Fs } { Fs.read(p) })     -- uses { Fs }
```

拡張形（自身も effect を発生させつつ callback の effect を足す）:

```emela
fn traced<T, U, e>(x: T, f: (T) -> U uses e) -> U uses { Log, ..e } {
    Log.info("call")     -- uses { Log }
    f(x)                 -- uses e
}
```

## Compilation Notes

この節は非規範的である．

- **単相化不要**: effect row は静的な契約で実行時表現を持たない（0009 / 0012）．したがって
  effect-row 変数は型検査後に消去するだけでよく，0014 の型単相化のような特殊化を要しない．
  型パラメータ `T` / `U` の単相化は従来どおり行い，effect row は erase する．同じ `map<T, U, e>`
  でも，`e` の違いだけで別の特殊化を生む必要はない（`T` / `U` が同じなら単一の特殊化を共有する）．
  mangling にも row は関与しない．
- typed IR（0012）は effect 注釈を保つが，現れるのは **宣言 row の具体部** であり，row 変数（tail）
  は erase される．特殊化を row で分けないため，呼び出しごとの具体化結果が IR に現れることもない．
- capability manifest（0025）は到達可能な platform 呼び出しから計算され（0049 D4 / D5），effect
  注釈に依存しない．row 変数の erase は権限表の計算に影響しない．

## Open Questions

- 複数 row 変数間の順序・重複の正規化と，row の等価判定．
- **error-row 多相**（`throws { A, ..r }` 的な行変数 / union error）の導入（0011 / 0014 の Open
  Questions と連動）．導入時は `<...>` 内の宣言とし，本仕様の row パラメータと同じ扱いとする．
- row 引数の明示指定構文（0014 の明示型引数と連動）．
- bounded row variable（`e ⊆ { Fs, Net }` のような上界付き row 変数）による緩和（0023 の Open
  Questions と共有）．
- 無名関数が **自身の** row パラメータを量化できるようにするか（0014 は無名関数の型パラメータを
  禁止．囲む関数の row パラメータの参照は本仕様で許可済み）．
- trait メソッド・effect operation・`extern fn` の row 多相．
