# 0030: Stdlib String Operations

Status: Draft

`String` の基本操作（`length` / `char_at` / `slice` / `chars`）と `Char.code`，および `String` /
`Char` の等価比較を定義する仕様．単位はすべて **Unicode scalar value**（0001 / 0007 / 0017）で統一する．
0021（Intrinsics）のパターンに従い，intrinsic を stdlib `std.string` が包む．

## Summary

- intrinsic `string_length` / `string_char_at` / `string_slice` / `string_eq` / `char_code` を追加し，
  `std.string` の `pub fn` が包む．
- 添字・長さ・範囲はすべて scalar 単位である (MUST)．byte 単位の API は提供しない．
- `impl Eq for String` / `impl Eq for Char` を Core Prelude（0021）に追加し，`==` / `!=`（0020 / 0027）
  を `String` / `Char` に使えるようにする．

## Motivation

0007 / 0017 の後も，文字列の長さ・要素取得・部分文字列が存在せず，`String` は生成（リテラル・`++`・
`from_char`）しかできなかった．また 0020 の `Eq` instance は `Int` / `Float` のみで，文字列比較が
書けなかった．本仕様は「初日に書ける」ための最小の文字列 API を，既に確定した scalar 単位
（0001 / 0007: 内部表現は UTF-8，可視単位は scalar）で与える．

## Specification

### intrinsic（0021 に従い宣言）

```emela
intrinsic fn string_length(s: String) -> Int uses {}
intrinsic fn string_char_at(s: String, i: Int) -> Char uses {}   -- 事前条件: 0 <= i < length
intrinsic fn string_slice(s: String, start: Int, end: Int) -> String uses {}
intrinsic fn string_eq(a: String, b: String) -> Bool uses {}
intrinsic fn char_code(c: Char) -> Int uses {}
```

- `string_length` は scalar 数を返す (MUST)．byte 数ではない．
- `string_char_at` の事前条件は `0 <= i < string_length(s)` であり，範囲外は panic する (MUST)．
  境界検査済みの安全な API は下記 wrapper が提供する．
- `string_slice(s, start, end)` は scalar 添字 `[start, end)` の部分文字列を返す．範囲は**クランプ**
  する (MUST): `start < 0` は `0` に，`end > length` は `length` に切り詰め，`start >= end` なら
  `""` を返す．panic しない．
- `string_eq` は scalar 列としての等価（＝ UTF-8 byte 列の等価と一致）を判定する．
- `char_code(c)` は `c` のコードポイント（0017 の `Char::from_code` の逆）を返す．

### std.string の公開 API

```emela
module string

pub fn length(s: String) -> Int uses {}                       -- string_length
pub fn char_at(s: String, i: Int) -> Option<Char> uses {}     -- 境界検査付き
pub fn slice(s: String, start: Int, end: Int) -> String uses {}
pub fn is_empty(s: String) -> Bool uses {}                    -- length(s) == 0
pub fn chars(s: String) -> Array<Char> uses {}                -- 全 scalar の配列
```

- `char_at` は範囲外で `None` を返す (MUST)（0011: 不在は `Option`）．panic する変種は提供しない．
- `chars` は `length` / `char_at` から純粋 Emela で実装できる．

### Eq instance の追加（Core Prelude）

- Core Prelude（0021）に次を追加する (MUST)．

```emela
impl Eq for String {
    fn eq(a: String, b: String) -> Bool uses {} { string_eq(a, b) }
}

impl Eq for Char {
    fn eq(a: Char, b: Char) -> Bool uses {} { char_code(a) == char_code(b) }
}
```

- これは 0020 の「Core Prelude の instance 集合」（`Eq`: `Int`, `Float`）を拡張する．`==` / `!=`
  （0027）が `String` / `Char` に対して import 無しで使える．
- `Ord for String`（辞書順）は本仕様では追加しない（Open Questions: 照合順序の設計を要する）．
  `Ord for Char` はコードポイント順として追加してよい (MAY)．

## Examples

```emela
import std.string.length
import std.string.char_at
import std.string.slice

fn initials(name: String) -> String {
    match char_at(name, 0) {
        Some(c) -> String::from_char(c)
        None -> ""
    }
}

fn main() -> Bool {
    let s = "héllo"
    length(s) == 5                 -- scalar 単位（byte なら 6）
        && slice(s, 1, 3) == "él"
        && initials(s) == "h"
}
```

## Compilation Notes

この節は非規範的である．

- 内部表現は UTF-8（0001）なので，`string_length` / `string_char_at` は素朴には O(n) である．
  ヘッダに scalar 数をキャッシュすれば `length` は O(1) になる（0024 のヘッダ設計）．`char_at` の
  O(1) 化が必要になれば，ASCII-only フラグや breadcrumbs 等の実装最適化で対応する（観測可能な意味は
  変わらない）．
- JavaScript backend: JS の `String` は UTF-16 なので，`length` / `char_at` / `slice` は**コード
  ポイント単位**の実装（`[...s]` 相当・サロゲートペア考慮）にしなければ WASM と意味がずれる．
  `s.length` をそのまま使ってはならない．
- `string_eq` は WASM では長さ比較＋ `memory.compare` 相当，JS では `a === b`．

## Open Questions

- `split` / `contains` / `starts_with` / `trim` / `to_upper` 等の拡充（Unicode 依存の操作は locale /
  tables の問題があるため慎重に）．
- `Ord for String` の照合順序（コードポイント順 vs 照合アルゴリズム）．
- grapheme cluster 単位の API（scalar 単位との使い分け）を将来提供するか．
- `String::from_char` の WASM 実装の多バイト UTF-8 対応（0017 の Open Question と共有．本仕様の
  `chars` / `slice` は多バイトを正しく扱う前提である）．
