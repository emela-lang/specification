## 0003: Function Definitions and Calls

Status: Draft

### 概要

関数定義，呼び出し，評価順，再帰，第一級関数の扱いを定義する仕様．

### 仕様

```emela
fn add(x: Int, y: Int) -> Int {
    x + y
}
```

Emela では関数は第一級関数である．よって，名前付き関数は以下のように変数に束縛されていると解釈できる:

```emela
let add: (Int, Int) -> Int uses {} = fn (x: Int, y: Int) -> Int uses {} {
    x + y
}
```

- `fn` で関数定義
- 引数は左から右に評価
- 関数本体は block expression
- 関数は lexical scope を持つ
- トップレベル関数の再帰は許可する
- 末尾再帰呼び出しが最適化されるかどうかは規定しない

### 関数型

関数型は引数型，戻り値型，`throws` 節 (error 型)，Effect を持つ．`throws` 節と error 処理の意味論は 0008, 0011 で定義する．

これは `Int` を2つ受取り，`Int` を返し，effect を発生させない関数型．

```emela
(Int, Int) -> Int uses {}
```

`uses {}` は純粋関数であることを表明していて，省略可能である．以下は等価:

```emela
(Int) -> Int uses {}

(Int) -> Int
```

これは Effect を発生させる関数型で，`fs` を発生させうる関数型である．

```emela
(Path) -> String uses { fs }
```

`uses` の後ろに書かれた Effect は型の一部として扱われるため，以下の関数型は違う型として扱われることを期待する．

```emela
(Path) -> String uses {}
(Path) -> String uses { fs }
```

ただし，row の部分集合方向の適合（`uses {}` の関数値を `uses { fs }` が期待される位置に渡す
subsumption）は spec 0023 が定める．

### 関数値

関数は値である．

```emela
fn add1(x: Int) -> Int uses {} {
    x + 1
}

let inc = add1
let res = add1(41) // res = 42
```

関数は関数の引数に渡すことができる．

```emela
fn apply(x: Int, f: (Int) -> Int uses {}) -> Int uses {} {
    f(x)
}

fn add1(x: Int) -> Int uses {} {
    x + 1
}

fn main() -> Int uses {} {
    apply(41, add1)
}
```

関数は関数の戻り値として返すことができる．関数を返す関数型は括弧で明確化する．

```emela
fn choose(flag: Bool) -> ((Int) -> Int uses {}) uses {} {
    if flag {
      add1
    } else {
      sub1
    }
}
```

### 無名関数・クロージャ

変数に束縛されることを前提とした無名関数を定義できる．

```emela
let add1 = fn (x: Int) -> Int uses {} {
    x + 1
}
```

構文は次の形．

```emela
fn (parameter_list) -> ReturnType uses EffectRow block
```

無名関数も通常の関数値であり，引数として渡せる．

```emela
fn apply(x: Int, f: (Int) -> Int uses {}) -> Int uses {} {
    f(x)
}

fn main() -> Int uses {} {
    apply(41, fn (x: Int) -> Int uses {} {
      x + 1
    })
}
```

無名関数は外側の lexical scope の値を capture できる．

```emela
fn makeAdder(n: Int) -> ((Int) -> Int uses {}) uses {} {
    fn (x: Int) -> Int uses {} {
      x + n
    }
}
```

この場合、返される関数値は n を capture する．

```emela
let add10 = makeAdder(10)
add10(5) // 15
```

### 関数の呼び出し

関数呼び出しは関数値に対する call として扱う．

```emela
f(a, b)
```

ここで f は名前付き関数でも，変数に束縛された関数値でもよい．

```emela
add(1, 2)
```

は，名前 add に束縛された関数値を呼び出す．

```emela
let f = add
f(1, 2)
```

も同じ．

### 評価順

```emela
f(g(), h())
```

これは，`g() -> h() -> f(result_of_g, result_of_h)` の順に評価される．

### 再帰呼び出し

名前付き関数は自分自身を参照できる．

```emela
fn fact(n: Int) -> Int uses {} {
    if n == 0 {
      1
    } else {
      n * fact(n - 1)
    }
}
```

トップレベル関数の相互再帰も可能．

```emela
fn even(n: Int) -> Bool uses {} {
    if n == 0 {
      true
    } else {
      odd(n - 1)
    }
}

fn odd(n: Int) -> Bool uses {} {
    if n == 0 {
      false
    } else {
      even(n - 1)
    }
}
```

### Effect と関数値

関数値の effect は，その関数を呼び出したときに発生しうる effect を表す．
```emela
fn applyPure(
    x: Int,
    f: (Int) -> Int uses {}
) -> Int uses {} {
    f(x)
}
```

`applyPure` は純粋関数だけ受け取れるので，以下のコードは有効である．

```emela
fn add1(x: Int) -> Int uses {} {
    x + 1
}

applyPure(1, add1)
```

以下のコードは無効である．

```emela
fn readThenAdd(x: Int) -> Int uses { fs } {
    ...
}

applyPure(1, readThenAdd)
```

effectful な関数を受け取りたい場合は，引数の関数型にも effect を書く．

```emela
fn applyFs(
    x: Int,
    f: (Int) -> Int uses { fs }
) -> Int uses { fs } {
    f(x)
}
```

より一般化するなら effect row に対するジェネリクスが必要になる．これは spec 0022 (Effect-Row Polymorphism) で定義する（effect-row 変数は sigil `'` を付け，型パラメータ `<>` とは別カテゴリとする）．

```emela
fn apply<T>(
    x: T,
    f: (T) -> T uses 'e
) -> T uses 'e {
    f(x)
}
```

### IR 上でのクロージャ表現

Emela では関数値は単一の値として扱う．

```emela
let f = makeAdder(10)
f(5)
```

IR では，関数値は概念的に次の2つを持つ．

- function pointer
- environment pointer

型としては次のように表せる。

```emela
FuncRef<(Int) -> Int effects {}>
```

または明示的に、

```emela
Closure {
    code: FunctionPointer
    env: EnvRef
}
```

```emela
fn makeAdder(n: Int) -> ((Int) -> Int uses {}) uses {} {
    fn (x: Int) -> Int uses {} {
        x + n
    }
}
```

これは以下の IR に変換される．

```
fn makeAdder(n: Int) -> FuncRef<(Int) -> Int effects {}> effects {} {
    %env = alloc.env { n: Int }
    env.store %env.n, n

    %closure = make_closure @makeAdder.lambdaO, %env
    return %closure
}

fn makeAdder.lambdaO(%env: EnvRef, x: Int) -> Int effects {} {
    %n = env.load %env.n
    %O = add.i32 x %n
    return %O
}

%f = call @makeAdder(10)
%r = call_closure %f(5)
```

トップレベル関数は environment を持たない closure として扱ってよい．

```
%f = make_closure @add1, null_env
```
