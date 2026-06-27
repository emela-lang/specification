# 0004: Struct Types

Status: Draft

## Summary

このドラフトは、Emela に名前付き product type として `struct` を導入します。
最初の対象は、ちょうど 1 個の名前付きフィールドを持つ単一フィールド struct です。

## Motivation

プリミティブ型だけでは、値の意味を型名として残せません。たとえばエラーコードを
単なる `I32` ではなく `Error` として扱うには、軽量な名前付き型が必要です。

## Specification

トップレベルには、関数定義に加えて `struct` 宣言を置けます。

```text
program =
  top_level_item*

top_level_item =
  struct_decl
  | function

struct_decl =
  struct type_name { struct_field }

struct_field =
  identifier : type

type =
  I32
  | Bool
  | Unit
  | type_name

expr =
  ...
  | type_name { identifier : expr }
  | expr . identifier
```

`type_name` は識別子です。このドラフトでは、型名と値名は別名前空間です。

`struct` はちょうど 1 個のフィールドを MUST 持ちます。複数フィールド struct は後続仕様で
導入します。

`struct` 値は `Name { field: value }` で構築します。`Name` は宣言済み struct 型で
なければなりません。`field` は宣言されたフィールド名と一致しなければなりません。
`value` は宣言されたフィールド型と一致しなければなりません。

単一フィールド struct のフィールドは `expr.field` で取り出せます。`expr` はその
フィールドを持つ struct 型でなければなりません。

```emela
struct Error {
  code: I32
}

fn main() -> I32 {
  error = Error { code: 1 }
  error.code
}
```

## Compilation Notes

初期コンパイラは、単一フィールド struct をそのフィールド値と同じ表現に lower できます。
この表現は非規範的です。複数フィールド struct の導入時に ABI を再定義できます。

## Open Questions

- 複数フィールド struct の構文、layout、ABI。
- フィールド名を struct 型ごとに名前解決するか、別の投影規則を導入するか。
