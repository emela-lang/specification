## 0008: Function Types with Effects

Status: Draft

関数型を effect row および `throws` 節付きで定義する仕様．

関数型は次の要素から構成される．

- 引数型
- 戻り値型 `T`
- 送出しうる error の型を表す `throws E` 節 (省略可能)
- 要求する capability / effect row を表す `uses E` 節 (省略可能)

### 型

完全形は次のとおり．

```text
(A, B) -> T throws E uses F
```

`throws E` は関数が送出しうる error の型を表す．`uses F` は関数が要求する capability / effect row を表す．

`throws` と `uses` は独立した channel である．error は `throws` で，capability は `uses` で追跡する．error 処理の意味論は 0011 で定義する．

### 省略

`throws` を省略した関数は error を送出しない (non-throwing)．

```text
A -> T uses F
```

`uses {}` の純粋関数は `uses {}` を省略しても良い．

```text
A -> T
```

組み合わせ:

```text
Path -> String                              // pure, non-throwing
Path -> String uses { fs }                  // effectful, non-throwing
Path -> String throws IoError               // throwing, pure
Path -> String throws IoError uses { fs }   // throwing, effectful
```

`throws E` と `uses F` はいずれも型の一部であり，public API の一部である (0010)．したがって次の3つは異なる型として扱う．

```text
Path -> String
Path -> String throws IoError
Path -> String throws IoError uses { fs }
```

### 高階関数における Effect 型

```emela
fn map<T, U>(
  xs: Array<T>,
  f: T -> U uses e
) -> Array<U> uses e
```

`throws` についても同様に，error 型を型変数として受け取り，伝播させることができる．

```emela
fn mapThrows<T, U, E>(
  xs: Array<T>,
  f: T -> U throws E
) -> Array<U> throws E
```
