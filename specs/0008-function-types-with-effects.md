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

なお，この省略規則は**関数型の表記**についてのものである．**関数定義**で `uses` 節を省略した場合の
意味（本体からの推論）は spec 0023 (Effect Inference and Subsumption) が定める．

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

effect row は **effect-row 変数**（row パラメータ）で受け取り，伝播できる．row パラメータは sigil の
ない小文字始まりの識別子で，型パラメータと同じ `<...>` に宣言する（`<...>` 内は先頭文字の大小で
型パラメータと振り分ける）．意味論は spec 0022 (Effect-Row Polymorphism) で定義する．

```emela
fn map<T, U, e>(
  xs: Array<T>,
  f: (T) -> U uses e
) -> Array<U> uses e
```

`throws` についても，error 型を型変数として受け取り，伝播させることができる．ここで `E` は通常の型
パラメータ（0014）である．`throws` は単一の error 型を取る（0011）ため，これは effect-row 変数ではなく
型変数であり，spec 0022 ではなく 0014 の対象である．両者は同じ `<...>` に同居する（`E` は大文字 = 型，
`e` は小文字 = row）．

```emela
fn mapThrows<T, U, E>(
  xs: Array<T>,
  f: (T) -> U throws E
) -> Array<U> throws E
```
