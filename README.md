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

- [0000: Language Design Principles](specs/0000-language-design-principles.md)
- [0001: Core Values and Types](specs/0001-core-values-and-types.md)
- [0002: Bindings and Mutation](specs/0002-binding-and-mutation.md)
- [0003: Function Definitions and Calls](specs/0003-function-definitions-and-calls.md)
- [0004: Blocks and Expression Semantics](specs/0004-blocks-and-expression-semantics.md)
- [0005: Enum and match expression](specs/0005-enum-and-match-expression.md)
- [0006: Records](specs/0006-records.md)
- [0007: Arrays and Strings](specs/0007-array-and-strings.md)
- [0008: Function Types with Effects](specs/0008-function-types-with-effects.md)
- [0009: Effect Semantics](specs/0009-effect-semantics.md)
- [0010: Modules, Imports, and Visibility](specs/0010-modules-imports-visibility.md)
- [0011: Error handling](specs/0011-error-handling.md)
- [0012: Typed IR](specs/0012-typed-ir.md)
- [0013: Platform Functions and Runtime Provision](specs/0013-platform-functions-and-runtime-provision.md)
- [0014: Generic Functions](specs/0014-generic-functions.md)
- [0015: If Expression](specs/0015-if-expression.md)
- [0016: Integer Division and Remainder](specs/0016-integer-division-and-remainder.md)
- [0017: Char and String Concatenation](specs/0017-char-and-string-concatenation.md)
- [0018: Qualified Imports and Calls](specs/0018-qualified-imports-and-calls.md)
- [0019: Pipeline Operator](specs/0019-pipeline-operator.md)
- [0020: Traits](specs/0020-traits.md)
- [0021: Intrinsic Functions](specs/0021-intrinsics.md)
- [0022: Effect-Row Polymorphism](specs/0022-effect-row-polymorphism.md)
- [0023: Effect Inference and Subsumption](specs/0023-effect-inference-and-subsumption.md)
- [0024: Memory Model](specs/0024-memory-model.md)
- [0025: Capability Manifest](specs/0025-capability-manifest.md)
- [0026: Embedder-Defined Capabilities](specs/0026-embedder-defined-capabilities.md)
- [0027: Boolean and Comparison Operators](specs/0027-boolean-and-comparison-operators.md)
- [0028: Generic Data Type Declarations](specs/0028-generic-data-types.md)
- [0029: Stdlib List](specs/0029-stdlib-list.md)
- [0030: Stdlib String Operations](specs/0030-stdlib-string.md)
- [0031: Stdlib Option Combinators](specs/0031-stdlib-option.md)
- [0032: Packaging and Dependency Resolution](specs/0032-packaging.md)
