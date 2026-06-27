# Emela Specification

Emela は Native と WebAssembly へのコンパイルを想定した、実験的な関数型言語です。

このリポジトリでは、言語仕様を小さな単位で段階的に定めます。各トピックはまず
小さな仕様ドラフトとして作成し、未解決の設計論点が解消されたものだけを安定化
します。

## Goals

- コア言語は、ゼロから実装できる程度に小さく保つ。
- 実行時の挙動は明示的に定義し、Native/WASM 間で移植可能にする。
- 言語境界では、型付きの関数型セマンティクスと明示的な effect tracking を優先する。
- 実装戦略より先に、観測可能な挙動を仕様化する。

## Non-Goals

- 最初のバージョンで大きな標準ライブラリを持つこと。
- コアセマンティクスが安定する前に高度な最適化規則を定めること。
- ターゲットごとにプログラムの意味が変わる挙動を許すこと。

## Specification Format

各仕様ファイルは次の構造を使います。

```md
# NNNN: Title

Status: Draft | Accepted | Superseded

## Summary

機能または規則の短い説明。

## Motivation

なぜ今この仕様が必要か。

## Specification

規範的な規則。必要に応じて MUST, MUST NOT, SHOULD, MAY を使う。

## Examples

小さなソース例と期待される意味。

## Compilation Notes

Native/WASM への lowering 制約、ABI、表現、実装上の注意。
明示しない限り、この節は非規範的な補足とする。

## Open Questions

Accepted にする前に解くべき未解決事項。
```

仕様ファイルは `specs/` に置き、ゼロ埋めの番号プレフィックスを付けます。
Accepted になった仕様は非互換に書き換えず、古い仕様を supersede する新しい仕様を
作成します。

## Current Drafts

- [0000: Minimal Core Language](specs/0000-minimal-core-language.md)
- [0001: Effect System](specs/0001-effect-system.md)
- [0002: Traits and Operators](specs/0002-traits-and-operators.md)
- [0003: Platform Capabilities](specs/0003-platform-capabilities.md)
- [0004: Struct Types](specs/0004-struct-types.md)
- [0005: Enum Types](specs/0005-enum-types.md)
- [0006: Result Type](specs/0006-result-type.md)
- [0007: Generic Types](specs/0007-generic-types.md)
- [0008: Imports](specs/0008-imports.md)
- [0009: Type Annotations](specs/0009-type-annotations.md)
