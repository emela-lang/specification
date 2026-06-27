# 0006: Result Type

Status: Draft

## Summary

このドラフトは、`struct` と `enum` に基づく `Result` 型の慣用形を定義します。
`Result` はこの段階では組み込み特殊型ではなく、通常の enum として宣言します。

## Motivation

失敗する可能性のある計算を effect marker だけで表すと、失敗値を通常の制御フローで扱えません。
`Result` は、成功と失敗をどちらも値として返し、`match` で明示的に処理するための基礎です。

## Specification

`Result` は、通常の enum として次の形で定義できます。

```emela
enum Result {
  Ok(I32)
  Err(Error)
}
```

`Ok` は成功値を持つ variant、`Err` は失敗値を持つ variant として扱う SHOULD です。
このドラフトではジェネリック型がないため、プログラムは用途ごとに具体的な payload 型を
持つ `Result` enum を宣言します。

失敗値は通常の型です。たとえば単一フィールド struct を使ってエラーコードを表せます。

```emela
struct Error {
  code: I32
}

enum Result {
  Ok(I32)
  Err(Error)
}

fn checked(value) -> Result {
  match value == 0 {
    true -> Err(Error { code: 1 })
    false -> Ok(value)
  }
}

fn main() -> I32 {
  match checked(0) {
    Ok(value) -> value
    Err(error) -> error.code
  }
}
```

`Result` を返す関数は、呼び出し側が `match` で `Ok` と `Err` の両方を処理できるように
する SHOULD です。wildcard arm で片方をまとめて処理することは可能です。

## Compilation Notes

`Result` の実行時表現は [0005: Enum Types](0005-enum-types.md) の enum 表現に従います。
`Result` 専用の ABI は定義しません。

## Open Questions

- `Result<T, E>` のようなジェネリック型の導入。
- `?` 演算子または早期 return 構文の導入。
- `Ok`/`Err` を標準 prelude として暗黙に提供するかどうか。
