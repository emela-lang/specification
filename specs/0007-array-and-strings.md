## 0007: Arrays and Strings

Status: Accepted

### String

```emela
let name = "alice"
```

- `String` は immutable である
- 内部表現は UTF-8 の byte 列（0001）
- ユーザーから見える要素と length の単位は **Unicode scalar value**（`Char`, 0017），byte ではない
- 添字・長さ・部分文字列はすべて scalar 単位で数える
- Emela では byte sequence ではなく，text として扱う

### Array

```emela
let xs = [1, 2, 3]
```

- `Array<T>` は固定長の配列
- 可変長配列に関しては別に定義する

基本操作は、spec 0021 のジェネリック **intrinsic 関数**（bare 名、stdlib の Core Prelude が宣言し backend が native 命令へインライン）として供給される。

```emela
intrinsic fn array_length<T>(a: Array<T>) -> Int uses {}
intrinsic fn array_get_unchecked<T>(a: Array<T>, i: Int) -> T uses {}
intrinsic fn array_push<T>(a: Array<T>, x: T) -> Array<T> uses {}
```

- `array_get_unchecked` は前提 `0 <= i < array_length(a)` を満たす添字を要求する（未検査の生アクセサ）。
- 安全な添字アクセスは stdlib の `pub fn` ラッパ `array_get<T>(a: Array<T>, i: Int) -> Option<T>` が提供する。境界外は `None` を返し panic しない（spec 0011: 不在は `Option`、panic する版はない）。`std.string.char_at` が `string_char_at` を包むのと同じ形。
- `array_push` は `x` を末尾に足した新しい配列を返す純粋コピーで、元の `a` は変更しない。

> **改訂（spec 0021）**: 当初これらは `Array::length` / `Array::get` / `Array::push`（あるいは `Array.get`）という型パス風のコンパイラ・ビルトインとして構想された。spec 0021（Intrinsics）で通常のジェネリック intrinsic 関数（bare 名）へ移され、`::` は enum variant 専用になった（spec 0018 R7）。安全な添字アクセス `array_get` は `Option<T>` を返す `pub fn` ラッパとし、生アクセサは `array_get_unchecked` とした。typed IR では専用ノードではなく汎用 `IrExpr::Intrinsic` として保持する。

