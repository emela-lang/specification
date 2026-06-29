## 0001: Core Values and Types

State: Draft

### 概要

値，型，ヒープ値，参照，null の有無を定義する最重要仕様．

### 型の初期セットとその分類

```text
Unit
Bool
Int
Float
String
Array<T>
Record
Enum
Function
```

このうち primitive 型として評価されるのは `Unit`, `Bool`, `Int`, `Float`．

`String`, `Array<T>`, `Record`, `Enum payload`, `Closure environment` はすべてヒープに格納される．

### 仕様の方針

- primitive 値はコピー可能
- heap 値はランタイムが管理する参照として扱う．
- `String` は `StringRef` ヒープ参照
- `Array<T>` は `ArrayRef<T>` ヒープ参照
- `Record` はヒープ上の immutable な record として扱う
- `Enum` は tag + payload として扱う
- null は持たない
- nullable な値は `Option<T>` で表す
- 所有権，借用，move semantics は導入しない．
    - 基本的に GC に近い仕組みでメモリを管理する

### Examples

```emela
name = "alice"
xs = [1, 2, 3]
```

型は以下のとおりになる．

```text
name: String
xs: Array<Int>
```

IR での表現は次のような参照型に変化する．

```text
%name: StringRef
%xs: ArrayRef<Int>
```

Emela 自体は `StringRef` をサポートせず，ユーザーから見える型は `String` にする．

### Int / Float

`Int` は `i32` に初期は固定する．

`Float` は `f64` に初期は固定する．

### Record

値的に等価比較できない．

### String

文字単位は `byte`．
