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

```emela
Array.get(xs, i) -> Option<T>
```

