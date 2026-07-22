# Emela Language Reference

このディレクトリは **現在の真実（current-state normative reference）** を記述する。
「今、言語がどう振る舞うか」を知るにはここだけを読めばよい。

## 二層モデル

Emela の仕様は2層に分かれる。目的が異なるので、読む場所も異なる。

| 層 | 場所 | 何を書くか | いつ読むか |
|---|---|---|---|
| **熟議ログ（RFC）** | [`../specs/`](../specs/) | ある変更を主張した時点の議論・動機・経緯。番号付き・append-only。supersede も差分表現もここでは正常。 | *なぜ* そうなったかを知りたいとき |
| **リファレンス** | `reference/`（このディレクトリ） | 現在の規範。トピック別・現在形・自己完結。supersede チェーンは持たない。 | *今どうか* を知りたいとき |

`specs/` は言語を学ぶために読むものではない。学ぶ・実装する・参照するのは `reference/` である。

## リファレンスの書き方（規律）

読解コストを下げるために、各章は次を守る。

1. **現在形・自己完結**。各文は「今どうか」だけを述べる。「以前どうで何を廃止したか」は書かない。変更履歴は `specs/` と [CHANGELOG](CHANGELOG.md) に隔離する。
2. **規範を先に、動機は後ろか `specs/` へ**。MUST / MUST NOT / SHOULD / MAY（RFC 2119）で規則を述べ、経緯の散文で薄めない。
3. **用語は一度だけ定義**。各章冒頭の用語集で定義し、以後は参照する。
4. **コアは推論規則・文法で**。型付け・row 計算の中核は散文より規則ブロックで書く（短く・曖昧さが少なく・将来の機械化の種になる）。
5. **参照は定義へ**。「§Effect rows を見よ」のような定義への参照はよい。「00XX を更新する」のような履歴を辿らせる参照は書かない。
6. **Provenance で出自を示す**。各章末に、その章が畳み込んだ RFC 番号を一箇所だけ列挙する。履歴は辿れるが、本文には混ぜない。

## 章立て

**基礎**

- [Values & Types](values.md) — 値の分類・`Int`/`Float`/`Char`/`String`/`Array`/`Bytes`・`Never`
- [Bindings & Functions](functions.md) — 不変束縛・第一級関数・クロージャ・自己末尾呼び出し
- [Expressions & Control Flow](expressions.md) — block / `if` / 論理・比較演算子 / pipeline / 優先順位
- [Data Types](data-types.md) — enum / `match` / record / ジェネリックデータ型 / `Option`
- [Generics](generics.md) — 型パラメータ・不透明性・推論・単相化

**型システムと境界**

- [Effects (`uses`)](effects.md) — capability / effect row・first-class effects・handler・discharge・権限表
- [Errors (`throws`)](errors.md) — error channel・`throw` / `?` / `try`–`catch`・`panic`・fallible platform 関数
- [Traits](traits.md) — trait・impl・境界・孤児規則・演算子 trait 化・`Monoid`・単相化 / erasure
- [Modules & Imports](modules.md) — モジュール単位 import・可視性・名前解決順位
- [Runtime Boundary](runtime-boundary.md) — platform 関数・intrinsic・Core Prelude・capability manifest・`host.*`

### 未着手（現状は `specs/` を参照）

- Memory Model — 非循環ヒープ・ARC（`specs/0024` / `specs/0048`）
- Compilation Model — typed IR・lang items・attributes（`specs/0012` / `specs/0041` / `specs/0039`）
- Stdlib — list / string / option / HTTP（`specs/0029`–`specs/0031` / `specs/0044` / `specs/0046`）
- Tooling — packaging / LSP / fmt / test（`specs/0032`–`specs/0035` / `specs/0040`）
- Backend Targets — `wasm-unknown` / `wasm-wasip2` 等（`specs/0052`）

各章は独立して読めることを保つ。リファレンスから `specs/` を **規範として** 参照してはならない（自己完結が原則）。`specs/` への参照は Provenance と、まだリファレンス化していない隣接トピックの暫定案内に限る。
