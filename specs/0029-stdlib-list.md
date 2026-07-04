# 0029: Stdlib List

Status: Draft

可変長の列を表す **`List<T>`** と，そのコア関数（`map` / `filter` / `fold` 等）を標準ライブラリ
`std.list` として定義する仕様．0028（Generic Data Types）と 0022/0023（effect-row 多相・推論）を
前提とする．固定長の `Array<T>`（0007）を置き換えるものではなく，補完する．

## Summary

- `List<T>` は stdlib が宣言する再帰的ジェネリック enum（cons list）である．コンパイラ組み込みでは
  ない．
- 要素数が実行時に決まる列・先頭への追加・再帰的な分解は `List<T>`，要素数固定・添字アクセスは
  `Array<T>`（0007）と使い分ける．
- コア関数は effect-row 多相（`'e`）で宣言し，純粋にも effectful にも同一定義で使える．
- すべて純粋 Emela で実装可能である（intrinsic / platform 関数を要しない）．

## Motivation

`Array<T>` は固定長であり（0007），「実行時に要素数が決まる列を作る」手段が言語に無かった．関数型
言語の中核であるリスト処理（構築しながらの再帰・先頭分解）には，immutable な cons list が最も自然で
ある: 先頭への追加が O(1)・共有が安全（immutable, 0001）・構築順により非巡回（0024 の不変条件を
保つ）．0028 によりユーザー定義の再帰的ジェネリック enum が書けるようになったため，`List<T>` は
特別な機構なしに stdlib の通常の宣言として与えられる．

## Specification

### 型

`std.list` は次を宣言する．

```emela
module list

pub enum List<T> {
    Nil
    Cons(T, List<T>)
}
```

`List<T>` の値は `Nil` か `Cons(head, tail)` であり，`match` で分解できる（0005 / 0028）．

### コア関数

`std.list` は少なくとも次の関数を提供する (MUST)．シグネチャは規範である．

```emela
pub fn length<T>(xs: List<T>) -> Int uses {}

pub fn head<T>(xs: List<T>) -> Option<T> uses {}
pub fn tail<T>(xs: List<T>) -> Option<List<T>> uses {}

pub fn prepend<T>(x: T, xs: List<T>) -> List<T> uses {}      -- Cons(x, xs)

pub fn reverse<T>(xs: List<T>) -> List<T> uses {}

pub fn append<T>(xs: List<T>, ys: List<T>) -> List<T> uses {}

pub fn map<T, U>(xs: List<T>, f: (T) -> U uses 'e) -> List<U> uses 'e
pub fn filter<T>(xs: List<T>, pred: (T) -> Bool uses 'e) -> List<T> uses 'e
pub fn fold<T, A>(xs: List<T>, init: A, f: (A, T) -> A uses 'e) -> A uses 'e

pub fn from_array<T>(xs: Array<T>) -> List<T> uses {}
pub fn to_array<T>(xs: List<T>) -> Array<T> uses {}
```

- `map` / `filter` / `fold` の関数引数は effect-row 変数 `'e` を取り（0022），結果の effect は渡された
  関数の effect になる．
- `fold` は左畳み込み（先頭から順に `f(acc, x)`）である (MUST)．評価順は列の順で決定的である．
- `head` / `tail` は空リストで `None` を返す．panic する変種は提供しない（0011: 不在は `Option`）．
- 順序保存: `map` / `filter` は入力の順序を保つ (MUST)．

### 実装制約

- 上記はすべて純粋 Emela で実装できなければならない (MUST)．intrinsic（0021）や platform 関数（0013）
  への依存を持たない．したがって `List` はすべてのバックエンドで自動的に利用可能である（0013 / 0021 の
  カバレッジ検査に影響しない）．

## Examples

```emela
import std.list.map
import std.list.filter
import std.list.fold

fn sum(xs: List<Int>) -> Int {
    fold(xs, 0, fn (acc: Int, x: Int) -> Int { acc + x })
}

fn main() -> Int {
    let xs = Cons(1, Cons(2, Cons(3, Nil)))
    xs |> map(fn (x: Int) -> Int { x * 2 })
       |> filter(fn (x: Int) -> Bool { x > 2 })
       |> sum                                    -- 10
}
```

再帰による定義（`map` の参照実装）:

```emela
pub fn map<T, U>(xs: List<T>, f: (T) -> U uses 'e) -> List<U> uses 'e {
    match xs {
        Nil -> Nil
        Cons(h, t) -> Cons(f(h), map(t, f))
    }
}
```

## Compilation Notes

この節は非規範的である．

- 表現は 0028 の通常の enum lowering（tag + payload 参照）であり，`List` 固有の IR ノードは無い．
- 非巡回（0024）なので RC で決定的に回収される．`tail` の共有（persistent data structure）は immutable
  ゆえに安全で，コピーを要しない．
- 深い再帰（`map` の参照実装等）はスタックを消費する．TCO の扱い（0003 は「規定しない」）が決まる
  まで，stdlib 実装は蓄積引数＋`reverse` などスタック安全な形を選ぶとよい．

## Open Questions

- リストリテラル構文（`[1, 2, 3]` は `Array`（0007）．`List` 用の糖衣を導入するか，`from_array` で
  足りるとするか）．
- `flat_map` / `zip` / `take` / `drop` / `range` などの拡充と，命名規約の確定．
- TCO 保証（0003 の未規定）を stdlib の再帰実装の前提にするか．
- `Eq` / `Show` のパラメータ付き instance（`impl<T: Eq> Eq for List<T>`, 0020）を stdlib で提供するか．
