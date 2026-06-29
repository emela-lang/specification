## 0011: Error handling

Status: Approved

回復可能な失敗を Result<T, E>, Option<T> で表現する．

回復不能な異常，例外は panic で処理する．

? operator は， Result / Option の短絡評価のための糖衣構文として導入する．

## Error の種類

Option<T>: 値が存在しない

Result<T, E>: 回復可能な失敗

panic: 回復不能な異常

## Option

Option<T> は値が存在しない可能性を表す組み込み enum

```emela
pub enum Option<T> {
    Some(T)
    None
}
```

```emela
fn get<T>(xs: Array<T>, index: Int) -> Option<T> uses {}

fn findUser(id: UserId) -> Option<User> uses {}
```

Option<T> は失敗の理由を持たない．理由が必要な場合は，Result<T, E> を用いる．


## Result

Result<T, E> は回復可能な失敗を表す組み込み enum

```emela
pub enum Result<T, E> {
    Ok(T)
    Err(E)
}
```

```emela
fn read(path: Path) -> Result<String, IoError> uses { fs }

fn parse(input: String) -> Result<Ast, ParseError> uses {}

fn decodeUser(input: String) -> Result<User, DecodeError> uses {}
```

## Panic

panic は組み込みの回復不能エラーである．

panic は現在の通常評価を終了し，プログラムを回復不能な異常終了へ移行させる．

panic は通常の値を返さず，型は bottom type 的に扱っても良い．

`Never` は任意の期待型に適合する．

```emela
fn head<T>(xs: Array<T>) -> T uses {} {
    match Array.get(xs, 0) {
        Some(x) -> x
        None -> panic("empty array")
    }
}
```

panic は通常のエラー処理に使ってはならない．

WASM / WAMR では panic は runtime trap, runtime-provided panic handler に lowering しても良い．

WAMR embedder は panic message をログに出して異常終了しても良いが，通常評価へ復帰してはならない．

## ? Operator

? は Result<T, E> または Option<T> に対する短絡評価 operator

IDEA: Unwrappable のような trait を実装して，? を使える型をユーザー側でも実装できるようにさせる？

? は match, 早期 return に対する糖衣構文である．

### Result

expr の型が Result<T, E> の場合，以下のコードは等価である．

```emela
let value = expr?

let value = match expr {
    Ok(v) -> v
    Err(e) -> return Err(e)
}
```

expr? を使う関数の戻り値の型は Result<U, E> である必要がある．

```emela
fn load(path: Path) -> Result<Config, IoError> uses { fs } {
    let text = read(path)?
    parseConfig(text)?
}

read(path): Result<String, IoError>
parseConfig(text): Result<Config, IoError>
```

なので， ? は IoError をそのまま返すことができる．

### Option
 
expr? 型が Option<T> の場合，以下のコードは等価である．

```emela
let value = expr?

let value = match expr {
    Some(v) -> v
    None -> return None
}
```

expr? を使う関数の戻り値の型は Option<U> である必要がある．

```emela
fn firstName(user: User) -> Option<String> uses {} {
    let profile = user.profile?
    profile.firstName
}
```

### No Implicit Conversion

Option<T> と Result<T, E> は暗黙的に変換をしない．

### Evaluation Order

? は postfix operator であり，対象の式を1回だけ評価する．

```emela
let x = f()?
```

1. f() を評価する
2. 結果が Ok(v) / Some(v) なら v を取り出す
3. 結果が Err(e) / None なら現在の関数から return する

```emela
fn combine() -> Result<Int, E> uses {} {
    let x = a()?
    let y = b()?
    Ok(x + y)
}
```

a() が Err を返した場合，b() は評価されない．

## IR Lowering

```emela
let text = read(path)?
```

```text
%r = call @read(path)

switch_enum %r {
    Ok => block_ok
    Err => block_err
}

block_ok:
    %text = enum_payload %r 0
    continue %text

block_err:
    %e = enum_payload %r 0
    %out = make_enum Err(%e)
    return %out
```

```emela
let x = maybe?
```

```text
%r = eval maybe

switch_enum %r {
    Some => block_some
    None => block_none
}

block_some:
    %x = enum_payload %r 0
    continue %x

block_none:
    %out = make_enum None
    return %out
```

```emela
panic("unreachable")
```

```text
%msg = const.string "unreachable"
call @__emela_panic(%msg)
unreachable
```

