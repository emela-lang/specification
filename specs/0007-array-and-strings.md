## 0007: Arrays and Strings

Status: Approved

### String

```emela
let name = "alice"
```

- `String` は immutable である
- UTF-8 encoded text として扱う
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

