# 0047: Monoid

Status: Draft

**結合的な二項演算 `combine` と，その単位元 `empty` を持つ型** の抽象 `trait Monoid` を導入する．
これにより「畳み込みの初期値を型自身から得る」総称コード — 典型的には **空集合でも全域な `sum` /
`concat_all`** — を書けるようにする．本仕様は spec 0020（Traits）を拡張し，0020 が dispatchability
規則で禁じていた **「戻り値位置にのみ `Self` を持つメソッド」** を，単位元メソッドに限って解禁する
（0020 Open Questions「加法単位元」「戻り値位置にのみ `Self`」の解消）．

## Summary

- `trait Monoid { fn combine(a: Self, b: Self) -> Self; fn empty() -> Self }` を定義する．`combine` は
  0020 の通常のメソッド（引数ディスパッチ），`empty` は **戻り値位置にのみ `Self`** を持つ．
- 0020 の dispatchability 規則を精密に緩和する: trait メソッドは `Self` を **パラメータ型または戻り値型**
  の少なくとも一方に含めばよい．`Self` がパラメータに現れれば従来どおり **引数ディスパッチ**，戻り値に
  のみ現れれば **戻り値ディスパッチ**（実装は呼び出し位置の **期待型** から決まる）．どちらにも現れない
  メソッドは依然として宣言できない．
- `empty()` の `Self` は **期待型から解決** する（新構文は導入しない）．解決の規律は spec 0028 の
  payload-less variant（`Nil` / `None`）と **同一**: 期待型が `Self` を確定できなければコンパイルエラー
  とする．generic 文脈では境界付き型変数 `T: Monoid` が期待型となり，`Self := T` で discharge する．
- `Monoid` は **演算子に結び付けない**（`combine` に記号を与えない）．境界としてのみ使う．
- コヒーレンス（孤児規則）・単相化・IR erasure は 0020 のまま．戻り値ディスパッチも単相化で静的に解決し，
  typed IR（0012）に trait・辞書・型変数は現れない．辞書渡しは要さない．
- 本仕様は **HKT を導入しない**．`Monoid` は kind `*` の単一型 `Self` を抽象化する通常の trait であり，
  spec 0000 の非目標（HKT / higher-rank / GADT）に一切抵触しない．

## Motivation

0020 の境界付き総称関数は「加算可能な集合を `+` で畳む」ことを可能にしたが，`sum` の **初期値** を型から
得る手段が無いため，空配列を扱えない．0020 の例の `sum<T: Add>` は戻り値を `Option<T>` にして「空なら
`None`」と誤魔化している．

```emela
-- 0020 現状: 空配列のために Option を返さざるを得ない
fn sum<T: Add>(xs: Array<T>) -> Option<T> uses {} { ... }
```

本当に欲しいのは **単位元 `empty`**（`Int` なら `0`，`String` なら `""`，`Money` なら `Money{cents:0}`）を
型から取り出し，空でも全域な畳み込みを書くことである．

```emela
-- 望ましい姿: 空配列でも全域
fn sum<T: Monoid>(xs: Array<T>) -> T uses {} { ... }   -- 空なら empty()
```

単位元 `empty : () -> Self` は引数を持たず，`Self` が戻り値位置にしか現れない．0020 はこれを
dispatchability 規則（`Self` はパラメータ型に出現必須）で **宣言禁止** としていた．ディスパッチを引数型から
決められないためである．しかし Emela は既に **同型の解決** を別の場所で行っている: payload の無い enum
variant（`Nil` / `None`）の型引数を **期待型** から決める規律（0028）である．本仕様はこの既存規律を単位元
メソッドへ適用し，辞書渡しも新構文も無しに `empty` を解禁する．

## Specification

本仕様は spec 0020 を拡張する．0020 の trait 宣言・impl・境界・孤児規則・単相化・IR erasure を継承し，
dispatchability 規則と解決規則のみを一般化する．0000 の supersede は不要である（HKT を導入しない）．

### `Monoid` trait

`Monoid` は Core Prelude（spec 0021 / 0038）が宣言する組み込み trait とする．

```emela
trait Monoid {
    fn combine(a: Self, b: Self) -> Self uses {}
    fn empty() -> Self uses {}
}
```

- **法則（非規範）**: 実装は次を満たすべきである (SHOULD)．コンパイラは検査しない．
  - 結合律: `combine(combine(a, b), c) == combine(a, combine(b, c))`
  - 単位元: `combine(empty(), a) == a` かつ `combine(a, empty()) == a`
- `Monoid` を **単独 trait** とする（`Semigroup` への分割・`Monoid: Semigroup` の trait 継承は導入しない
  → Open Questions）．

### dispatchability 規則の緩和（0020 の精密化）

0020 は「各 trait メソッドはいずれかの **パラメータ型** の中に `Self` が出現しなければならない」と規定した．
本仕様はこれを次に置き換える (MUST)．

- trait の各メソッドは，`Self` を **パラメータ型または戻り値型** の少なくとも一方（ネストを含む）に
  含まなければならない．
  - `Self` が **いずれかのパラメータ型** に現れるメソッド（`combine`，および 0020 の全既存メソッド）は
    **引数ディスパッチ**．挙動は 0020 と完全に同一（後方互換）．
  - `Self` が **戻り値型にのみ** 現れるメソッド（`empty`，`fn zero() -> Self`，`fn parse(s: String) ->
    Self` 等）は **戻り値ディスパッチ**．実装は呼び出し位置の期待型から決まる（次項）．
  - `Self` が **どこにも現れない** メソッドは依然として宣言できない (MUST NOT)．
- 一つのメソッドで `Self` がパラメータと戻り値の両方に現れる場合は引数ディスパッチとする（パラメータ側が
  優先）．

### 戻り値ディスパッチの解決（期待型）

戻り値ディスパッチのメソッド呼び出し（`empty()`）は，`Self` を **期待型** から解決する (MUST)．

- **期待型の供給元**: 束縛の型注釈（`let x: Money = empty()`），関数引数位置のパラメータ型，関数の戻り値型，
  `match` の他 arm の型，`combine` など同一式中で `Self` を共有する隣接オペランドの型，`if` の他分岐の型．
  これらは 0028 が payload-less variant の型引数を確定するために既に用いる供給元と同一である．
- **generic 文脈**: 期待型が境界付き型変数 `T`（`T: Monoid` が in-scope）のとき，`Self := T` とし，境界
  `T: Monoid` により discharge 済みとみなす（0020 の bound 伝播）．具体化は単相化時に `T = C` が定まって
  から行う．
- **解決不能はエラー (MUST)**: 期待型から `Self` を一意に確定できない場合はコンパイルエラーとする．
  診断は 0028 の「推論で一意に定まらない」と同系統とし，型注釈を促す（例: `` `empty` の `Self` を
  決められない．束縛や引数に型注釈を付けてください ``）．文脈皆無の `let x = empty()` はこれに該当し，
  `let x: Money = empty()` と書けば解決する．
- 期待型で `Self = C`（具体型）が定まった後，コンパイラは 0020 と同じく境界を discharge する
  （`impl Monoid for C` の in-scope 存在を検査し，無ければエラー）．

### impl・境界・コヒーレンス

- `impl Monoid for Type { fn combine(...) {...}  fn empty() -> Type {...} }` は 0020 の impl 規則
  （シグネチャ厳密一致・網羅性・孤児規則・大域一意）にそのまま従う．`empty` の impl 側シグネチャは
  `Self := Type` を代入した `fn empty() -> Type uses {}` である．
- 境界 `<T: Monoid>` は 0020 の境界構文・伝播・discharge 規則をそのまま用いる．
- 孤児規則により (Monoid, 型) の実装は大域一意なので，戻り値ディスパッチでも `Self = C` が決まれば
  実装は一意に定まる（曖昧は生じない）．

### 演算子 trait との関係（設計方針）

`Monoid` は 0020 の演算子 trait（`Add` / `Sub` / `Mul` / `Div` / `Rem`）と **意図的に直交な層** とする．

- `combine` / `empty` は **いずれの演算子にも desugar しない** (MUST NOT)．`++`（Concat, 0017/0020）や
  `+`（Add）は従来どおりそれぞれの trait のメソッドであり，`Monoid` とは独立である．`Monoid` は総称関数の
  **境界としてのみ** 用いる（`sum<T: Monoid>` 等）．
- **演算子 trait を Monoid で置き換えたり，Monoid から導出したりはしない**．演算子 trait は「型がその記号の
  演算を持つ」という **能力** の層，`Monoid` は「結合演算＋単位元」という **代数構造** の層であり，両者は別物
  である．具体的には:
  - `Sub` に対応する Monoid は **存在しない**．減算は結合的でなく単位元も持たない（`a - b = a + (-b)` は
    加法群の **逆元** から導かれる派生演算）．`Sub` は演算子 trait のまま据え置く．
  - `Mul` は `Add` とは **別の** モノイド（単位元 `1`）である．ゆえに数値型は加法・乗法の 2 つのモノイドを
    持ち，(Monoid, 型) 大域一意（0020 孤児規則）の下では正準を選べない（→ 組み込み instance を保留する
    理由．Open Questions）．
- この方針は Rust std（演算子＝trait）＋ `num-traits`（`Zero` / `One` を演算とは別 trait に置く）と同型で
  ある．`Semigroup → Monoid → Group → Ring → Field` の **代数階層**（Scala spire・Swift
  `AdditiveArithmetic`・Idris 等）へ進む道は，0020 Open Questions の **trait 継承（スーパークラス）** の
  解決を前提とするため，本仕様では採らない（→ Open Questions）．

### 組み込み instance

- Core Prelude は `impl Monoid for String { combine = string_concat;  empty = "" }` を提供する (MUST)．
  文字列連結は結合的で単位元 `""` が一意なため曖昧が無い．
- **数値型（`Int` / `Float`）への組み込み instance は本仕様では提供しない**．加法（単位元 `0`）と乗法
  （単位元 `1`）のどちらを正準とするかが曖昧なためである（→ Open Questions）．数値を畳みたい利用者は
  当面 record / newtype に `impl Monoid` を与える（下の `Money` 例）．
- stdlib の `List<T>`（0029）は所有モジュール側で `impl<T> Monoid for List<T> { combine = append;
  empty = Nil }` を与えてよい (MAY)．孤児規則上，型の所有モジュールなので合法である．

## Examples

ユーザー record への impl（`Money` を空配列でも畳める）:

```emela
record Money {
    cents: Int
}

impl Monoid for Money {
    fn combine(a: Money, b: Money) -> Money uses {} {
        Money { cents: a.cents + b.cents }
    }
    fn empty() -> Money uses {} {
        Money { cents: 0 }
    }
}

fn sum<T: Monoid>(xs: Array<T>) -> T uses {} {
    fold_combine(xs, 0, empty())     -- empty() の Self は戻り値位置 T（sum の戻り値）から解決
}

fn fold_combine<T: Monoid>(xs: Array<T>, i: Int, acc: T) -> T uses {} {
    match Array.get(xs, i) {
        None -> acc
        Some(x) -> fold_combine(xs, i + 1, combine(acc, x))
    }
}

fn main() -> Int uses {} {
    let empty_total = sum([])                                  -- T = Money, empty() = Money{cents:0}
    let total = sum([Money { cents: 100 }, Money { cents: 50 }])
    empty_total.cents + total.cents                           -- 0 + 150 = 150
}
```

期待型からの `empty` 解決（注釈で `Self` を与える）:

```emela
let z: Money = empty()          -- OK: Self = Money を注釈から解決
-- let z = empty()              -- エラー: Self を決められない（注釈を促す）
```

文字列の単位元（組み込み instance）:

```emela
fn concat_all(xs: Array<String>) -> String uses {} {
    -- 空配列なら empty() = ""，非空なら combine で連結
    fold_combine_str(xs, 0, empty())    -- Self = String（戻り値型から）
}
```

## Compilation Notes

この節は非規範的な実装上の補足である．

- **戻り値ディスパッチの解決経路**: 参照実装は，payload-less variant（`Nil` / `None`）の型引数を
  `Type::Never` の穴として残し assignability で期待型から後決めする既存機構
  （`crates/emela/src/typecheck.rs` の enum 構築，`Type::Never` プレースホルダ）を，戻り値ディスパッチの
  メソッド呼び出しに一般化する．`register_traits` の dispatchability 検査を「`Self` がパラメータ **または
  戻り値** に出現」へ緩和し，戻り値ディスパッチのメソッドを `TraitMethodInfo` にフラグ付けする．
  `dispatch_method` は引数から `Self` が埋まらないとき期待型（呼び出し位置に伝播）から `Self` を決める．
- **単相化**: 期待型解決後の `empty()` 呼び出しは，`Self = C` が確定した時点で
  `impl Monoid for C` の `empty` メソッドへの **直接 call** に置換し，特殊化キューに積む（0020 と同一）．
  マングリングは 0020 の `Trait__Type__method` 規則をそのまま用いる（例 `Monoid__Money__empty`，
  `Monoid__String__combine`）．
- **IR erasure（0012 / 0020 不変条件）**: 戻り値ディスパッチも型検査・lowering で完全に解決されるため，
  typed IR には trait・辞書・型変数は現れない．backend は具体型の直接 call だけを受け取り，本仕様のための
  専用 IR ノードは追加しない．
- **generic 内の `empty()`**: `sum<T: Monoid>` 内の `empty()` は型検査時に `Self = T`（型変数）のまま残り，
  `sum` の各特殊化（`T = Money` 等）で具体化されて `Monoid__Money__empty` へ lower する．

## Open Questions

- **`Semigroup` への分割と trait 継承**: `trait Semigroup { fn combine }` を切り出し
  `trait Monoid: Semigroup { fn empty }` とするか．これは 0020 Open Questions の「trait 継承 /
  スーパークラス」の解決を要する．本仕様は単独 `Monoid` としてこれを回避した．
- **数値の Monoid（加法 vs 乗法）**: `Int` / `Float` に組み込み instance を与えるか．与えるなら加法
  （単位元 `0`）と乗法（単位元 `1`）の曖昧を，Haskell の `Sum` / `Product` newtype 相当で分離するか，
  加法を正準とするか．本仕様は組み込み instance を保留した．
- **一つの型に複数の Monoid**: 上記に関連し，`min` / `max` / 和 / 積 など同一型上の複数の Monoid を
  孤児規則の大域一意性の下でどう表すか（newtype による分離が有力）．
- **算術の代数階層**: `Sub` / `Mul` / `Div` を原理的に位置づける階層（`Group` の逆元＝減算，乗法モノイド，
  `Ring` / `Field`；あるいは Swift `AdditiveArithmetic` 相当の `+ - zero` 束ね）を導入するか．これは
  0020 の trait 継承（スーパークラス）の解決を前提とし，かつ `Int` の切り捨て除算・0 除算 trap（0016）が
  群・体の法則を満たさない点への配慮を要する．本仕様は演算子 trait を据え置き，この階層を将来課題とする．
- **戻り値ディスパッチの一般化**: `fn parse(s: String) -> Self`（戻り値のみ `Self`）など，`empty` 以外の
  戻り値ディスパッチメソッドを本仕様の期待型解決でどこまで許すか．本仕様の規則は既に一般（戻り値のみ
  `Self` を許可）だが，`parse` のように期待型が乏しい用途では明示型引数（下記）が要るかもしれない．
- **明示型引数（turbofish）**: 期待型が無い文脈でも `empty::<Money>()` と書けるようにするか．これは 0014
  Open Questions の明示型引数の導入を要する．本仕様は期待型解決で足りるため見送った．
- **derive**: `derive(Monoid)` によるフィールドごとの機械的な instance 生成（record の各フィールドが
  `Monoid` なら record も `Monoid`）．
- **`Foldable` / 総称 `fold`**: `Array` 以外のコンテナも畳める総称 `fold`．コンテナの抽象化は HKT を
  要するため（0000 非目標），本仕様の範囲外である．
