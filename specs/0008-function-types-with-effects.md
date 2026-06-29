## 0008: Function Types with Effects

Status: Draft

関数型を effect row 付きで定義する仕様．

### 型

```text
A -> B uses E

(A, B) -> C uses E
```

純粋関数は

```text
A -> B uses {}
```

純粋関数の場合は，`uses {}` を省略しても良い．

```text
A -> B
```

### 高階関数における Effect 型

```emela
fn map<T, U>(
  xs: Array<T>,
  f: T -> U uses e
) -> Array<U> uses e
```
