## 0000: Language Design Principles

State: Approved

### 概要

Emela の仕様全体の設計原則を定義する．

具体的な構文よりも仕様をどのようにして分けるかを定める．

### 決めること

- 表面構文は人間が書く契約である。
- IR はコンパイラが保証する意味である。
- Runtime Boundary は外界との副作用契約である。
- 構文糖衣は Core Semantics に影響してはならない。
- Native/WASM/WAMR で観測可能な意味が変わってはならない。
- 最初のバージョンでは ownership/borrowing を入れない。
- 最初のバージョンでは async/trait/macro/operator overloading を入れない。

### 結論

初期 Emela は次のモデルにする。

```text
strict expression-oriented functional core
+ immutable bindings by default
+ runtime-managed heap references
+ effect row in function types
+ capability-based runtime boundary
```

