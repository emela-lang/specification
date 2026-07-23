# 0031: Stdlib Option Combinators

Status: Draft

`Option<T>` を値として取り回すためのコンビネータ（`map` / `and_then` / `unwrap_or` / `ok_or` 等）を
標準ライブラリ `std.option` として定義する仕様．0011 が「Option には `?` を使わない」と定めた際に
前提としたコンビネータ群（0011 が `Option.map` / `Option.and_then` / `Option.unwrap_or` /
`Option.ok_or` を前方参照している）を正式化する．

## Summary

- `std.option` は `Option<T>` の標準コンビネータを提供する．すべて純粋 Emela（`match`）で実装可能．
- `ok_or` は不在 (`None`) を error channel（`throws`, 0011）へ**明示的に**橋渡しする唯一の標準手段で
  ある．
- 関数を取るコンビネータは effect-row 多相（row パラメータ `e`, 0022）で宣言する．

## Motivation

0011 は `?` を throwing な呼び出し専用とし，`Option` は「`match` またはコンビネータで扱う」と定めた．
コンビネータが無ければ `Option` の連鎖が `match` の入れ子になり，この決定が成立しない．また
「不在を回復可能な失敗として伝播したい」ユースケースには，`Option` → `throws` の明示的な橋
（`ok_or`）が要る（0011 の No Implicit Conversion の但し書きが参照する関数）．

## Specification

`std.option` は少なくとも次を提供する (MUST)．シグネチャは規範である．

```emela
module option

pub fn map<T, U, e>(o: Option<T>, f: (T) -> U uses e) -> Option<U> uses e

pub fn and_then<T, U, e>(o: Option<T>, f: (T) -> Option<U> uses e) -> Option<U> uses e

pub fn unwrap_or<T>(o: Option<T>, default: T) -> T uses {}

pub fn or<T>(o: Option<T>, alt: Option<T>) -> Option<T> uses {}

pub fn ok_or<T, E>(o: Option<T>, e: E) -> T throws E uses {}

pub fn is_some<T>(o: Option<T>) -> Bool uses {}
pub fn is_none<T>(o: Option<T>) -> Bool uses {}
```

- **`map`**: `Some(v)` なら `Some(f(v))`，`None` なら `None`．`f` は `o` が `Some` のときのみ評価
  される (MUST)．
- **`and_then`**: `Some(v)` なら `f(v)`，`None` なら `None`．`Option` を返す計算の連鎖（flatMap）．
- **`unwrap_or`**: `Some(v)` なら `v`，`None` なら `default`．`default` は正格評価される（呼び出し
  規約 0003 のとおり，常に評価される）．遅延させたい場合は将来の `unwrap_or_else`（Open Questions）．
- **`or`**: `Some` ならそのまま，`None` なら `alt`．
- **`ok_or`**: `Some(v)` なら `v` を返し，`None` なら `throw e` する．`Option` を error channel へ
  昇格する明示的な橋である．暗黙変換は存在しない（0011）ため，これが標準の変換点になる．
- **`is_some` / `is_none`**: 判定．
- `None` で panic する `unwrap` は**提供しない** (MUST NOT)．回復不能扱いにしたい場合は，呼び出し側が
  `match` して `panic` を書く（0011: panic は明示的な最終手段であり，標準 API が不在を panic に
  変換する近道を提供しない）．

### 実装制約

- すべて純粋 Emela で実装できなければならない (MUST)（`ok_or` は `throw` を用いる）．intrinsic /
  platform 関数への依存を持たず，全バックエンドで利用可能である．

参照実装:

```emela
pub fn map<T, U, e>(o: Option<T>, f: (T) -> U uses e) -> Option<U> uses e {
    match o {
        Some(v) -> Some(f(v))
        None -> None
    }
}

pub fn ok_or<T, E>(o: Option<T>, e: E) -> T throws E uses {} {
    match o {
        Some(v) -> v
        None -> throw e
    }
}
```

## Examples

`match` の入れ子をコンビネータとパイプ（0019）で平らにする:

```emela
import std.option.and_then
import std.option.unwrap_or

fn city_name(user: User) -> String {
    user.profile
        |> and_then(fn (p: Profile) -> Option<Address> { p.address })
        |> and_then(fn (a: Address) -> Option<String> { a.city })
        |> unwrap_or("unknown")
}
```

不在を error へ昇格して `?` で伝播（0011 の例の再掲）:

```emela
import std.option.ok_or

fn require_profile(user: User) -> Profile throws NotFound {
    ok_or(user.profile, NotFound)?
}
```

## Compilation Notes

この節は非規範的である．

- 通常の stdlib 関数であり，専用の IR ノードや backend 対応は不要．単相化（0014）とインライン化で
  `match` 直書きと同等のコードになることが期待できる．

## Open Questions

- `unwrap_or_else<T, e>(o, f: () -> T uses e)`（既定値の遅延評価版）等の拡充．
- `Option<Option<T>>` の `flatten`．
- `to_list` / `to_array` などコレクションとの相互変換（0029 と連動）．
- `Eq` のパラメータ付き instance（`impl<T: Eq> Eq for Option<T>`, 0020）を stdlib で提供するか
  （0029 の List と共通の課題）．
