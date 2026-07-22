# Reference Changelog

[リファレンス](README.md)（現在の真実の層）の **意味的変更** を時系列で記録する。
各章は現在形・自己完結で書かれ履歴を持たないため、「何がいつ変わったか」はここに集約する。

- 記録するのは規範の変化（規則の追加・変更・削除）である。字句修正・リンク調整は記録しない。
- 各エントリは、変更した章と、対応する [RFC（`specs/`）](../specs/) を示す。
- 詳細な経緯・却下案・動機は RFC 側にある。ここは索引である。

## 2026-07-22

- **基礎5章を新設**。[Values & Types](values.md)（`specs/0001` / `0007` / `0016` / `0017` / `0051`）、[Bindings & Functions](functions.md)（`specs/0002` / `0003` / `0045`）、[Expressions & Control Flow](expressions.md)（`specs/0004` / `0015` / `0027` / `0019`）、[Data Types](data-types.md)（`specs/0005` / `0006` / `0028` / `0042`）、[Generics](generics.md)（`specs/0014`）。
- **二層構成を導入**。`specs/`（熟議ログ）と `reference/`（現在の真実）を分離。「Accepted は supersede 必須」の旧ルールを廃止し、Accepted 仕様の直接改訂を許可（[README](README.md)）。
- **[Effects](effects.md)** を新設。`specs/0008` / `0009` / `0022` / `0023` / `0036` / `0037` / `0049` / `0050` を現在形に畳み込み。
- **[Errors](errors.md)** を新設。`specs/0011` / `0043`。
- **[Modules & Imports](modules.md)** を新設。`specs/0037`（`0010` / `0018` を置換）/ `0010`。
- **[Traits](traits.md)** を新設。`specs/0020` / `0047` / `0021`。
- **[Runtime Boundary](runtime-boundary.md)** を新設。`specs/0013` / `0021` / `0043` / `0025` / `0026`。
