# 0005: Enum Types

Status: Draft

## Summary

このドラフトは、Emela に名前付き sum type として `enum` を導入します。
最初の対象は、0 個または 1 個のペイロードを持つ variant です。

## Motivation

分岐する値を型で表すには sum type が必要です。`Bool` は固定された 2 分岐ですが、
ユーザー定義 enum はドメイン固有の分岐を表せます。

## Specification

トップレベルには、関数定義と `struct` 宣言に加えて `enum` 宣言を置けます。

```text
program =
  top_level_item*

top_level_item =
  enum_decl
  | function
  | ...

enum_decl =
  enum type_name { enum_variant* }

enum_variant =
  identifier
  | identifier ( type )

expr =
  ...
  | variant_name
  | variant_name ( expr )

pattern =
  ...
  | variant_name
  | variant_name ( pattern )
  | identifier
```

`enum` は 1 個以上の variant を MUST 持ちます。variant 名は同じプログラム内で一意で
なければなりません。この制約により、このドラフトでは `Ok(value)` のように型名なしで
variant constructor を書けます。

ペイロードなし variant は `Variant` と書きます。ペイロード付き variant は
`Variant(value)` と書きます。`value` は variant に宣言されたペイロード型と一致しなければ
なりません。

`match` pattern の `Variant` はペイロードなし variant に一致します。`Variant(pattern)` は
ペイロード付き variant に一致し、内側の pattern を payload に適用します。

`Variant(name)` の内側が通常の識別子である場合、その payload を新しいローカル束縛として
導入します。`Variant(_)` は payload を捨てます。

```emela
enum Switch {
  On
  Off
}

fn main() -> I32 {
  match On {
    On -> 1
    Off -> 0
  }
}
```

enum に対する `match` は、wildcard arm がない場合、すべての variant を網羅しなければ
なりません。すべての arm の式は同じ型を MUST 持ちます。

## Compilation Notes

ペイロードなし enum は integer tag として lower できます。

単一ペイロード enum は、非規範的には下位 bit に tag、残りに payload を置く 1 ワード値へ
lower できます。正確な layout は安定 ABI ではありません。

## Open Questions

- variant 名を enum 型ごとの名前空間へ移す時期。
- 複数ペイロード variant の構文と layout。
- enum に対する `==` の導入条件。
