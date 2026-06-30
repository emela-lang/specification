## 0011: Error handling

Status: Draft

回復可能な失敗は，関数型の `throws E` 節と `throw` 式で表現する．

値の不在は `Option<T>` で表現する．

回復不能な異常は `panic` で処理する．

`?` operator は throwing な呼び出しの error を呼び出し元へ伝播する短絡評価 operator として導入する．

`try` / `catch` 式は送出された error を捕捉してハンドルする．

## Error の種類

`throws E`: 回復可能な失敗 (error channel)

`Option<T>`: 値が存在しない

`panic`: 回復不能な異常

回復可能な失敗を `Result<T, E>` で表現することはしない．`Result` は組み込み型ではない．

## throws と throw

### throws 節

関数が送出しうる error の型は，戻り値型に続く `throws E` 節で宣言する (0008)．

```emela
fn read(path: Path) -> String throws IoError uses { fs }

fn parse(input: String) -> Ast throws ParseError uses {}
```

- `throws E` を持たない関数は error を送出しない．これは `throws Never` と等価である．`Never` 型の値は存在しないため，`throws Never` を宣言する関数は error を送出しえない．
- `E` は単一の型である．複数種類の error を扱う場合は enum を用いる．
- `throws` は `uses` とは独立した channel であり，capability を `throws` に書くことはできない．

```emela
enum LoadError {
    NotFound
    Decode(DecodeError)
}

fn load(path: Path) -> Config throws LoadError uses { fs }
```

### throw 式

`throw e` は error `e` を送出する．

```emela
throw IoError.NotFound
```

- `throw e` は通常の値を返さない．型は `Never` であり，任意の期待型に適合する．
- `throw e` が現れる関数は `throws E` を宣言していなければならず，`e` の型は `E` に適合していなければならない．ただし `throw e` が `try` block 内にある場合は，その error は `catch` に捕捉される (後述)．
- `throw` は capability ではないため，`uses` には現れない．

```emela
fn lookup(id: UserId, table: Table) -> User throws NotFound uses {} {
    match Table.get(table, id) {
        Some(user) -> user
        None -> throw NotFound
    }
}
```

## throwing な式の扱い

`throws E` を持つ関数の呼び出しを **throwing な呼び出し** と呼ぶ．throwing な呼び出しの成功値の型は `T` であるが，error を素通りさせることはできない．

throwing な呼び出しは，次のいずれかの位置に置かなければならない．これを満たさない throwing な呼び出しはコンパイルエラーとする．

1. `?` を付けて error を呼び出し元へ伝播する．
2. `try` block 内に置き，`catch` で捕捉する．

## ? Operator

`?` は throwing な呼び出しの error を，現在の関数の `throws` 節へ伝播する短絡評価 operator である．

`?` は throw, 早期 return に対する糖衣構文と捉えてよい．

### throws に対する ?

`expr` が `T throws E` を持つ throwing な呼び出しの場合，以下のコードは等価である．

```emela
let value = expr?

let value = try {
    expr
} catch {
    e -> throw e
}
```

`expr?` を使う関数の戻り値の型は `throws E` を宣言していなければならない．

```emela
fn load(path: Path) -> Config throws IoError uses { fs } {
    let text = read(path)?
    parse(text)?
}

read(path): String throws IoError uses { fs }
parse(text): Config throws IoError uses {}
```

`read` と `parse` の error 型は `IoError` であり，`load` の `throws IoError` と一致するため，`?` はそのまま伝播できる．

### error 型の一致

`?` が伝播する error 型は，現在の関数が宣言する error 型と一致していなければならない．

```emela
fn load(path: Path) -> Config throws LoadError uses { fs } {
    let text = read(path)?    // read は throws IoError なので reject
    parse(text)?
}
```

error 型が異なる場合は明示的に変換する．

```emela
fn load(path: Path) -> Config throws LoadError uses { fs } {
    let text = try {
        read(path)
    } catch {
        e -> throw LoadError.Io(e)
    }
    parse(text)?
}
```

後続仕様で `From` / `Into` 的な error 変換を導入してもよい．

### Option に対する ?

`?` は `Option<T>` に対しても使える．`expr` が `Option<T>` の場合，以下のコードは等価である．

```emela
let value = expr?

let value = match expr {
    Some(v) -> v
    None -> return None
}
```

`expr?` を使う関数の戻り値の型は `Option<U>` でなければならない．

```emela
fn firstName(user: User) -> Option<String> uses {} {
    let profile = user.profile?
    profile.firstName
}
```

`?` の意味は対象式の型によって決まる．throwing な呼び出しに対しては error を `throws` へ伝播し，`Option<T>` に対しては `None` を伝播する．両者は型により一意に区別される．

### No Implicit Conversion

`throws E` の error channel と `Option<T>` は暗黙的に変換しない．`Option<T>` を返す関数に対して `?` を使っても error は送出されず，throwing な呼び出しに対して `?` を使っても `None` は伝播しない．

### Evaluation Order

`?` は postfix operator であり，対象の式を1回だけ評価する．

```emela
let x = f()?
```

1. `f()` を評価する
2. 結果が成功値 / `Some(v)` なら値を取り出す
3. error が送出された / `None` なら現在の関数から短絡する

```emela
fn combine() -> Int throws E uses {} {
    let x = a()?
    let y = b()?
    x + y
}
```

`a()` が error を送出した場合，`b()` は評価されない．

### ? and Effects

`?` 自体は capability effect を発生させない．`expr?` の effect は `expr` の effect と同じである．

```emela
let text = read(path)?
```

`read(path): String throws IoError uses { fs }` の場合，この式全体も `uses { fs }` を要求する．

## try / catch 式

`try` / `catch` は送出された error を捕捉してハンドルする式である．

```emela
let cfg = try {
    load(path)
} catch {
    IoError.NotFound -> defaultConfig()
    e -> panic("unrecoverable")
}
```

### 意味論

- `try { block } catch { arms }` は式である．
- `block` を評価する．`block` が成功値 `v` で完了した場合，`try` / `catch` 式全体の値は `v` であり，`catch` は評価されない．
- `block` の評価中に error `e` が送出された場合，`e` を `catch` の arm に対して上から順に pattern match する．一致した arm の本体を評価し，その値が式全体の値となる．
- `try` block 内では throwing な呼び出しに `?` を付ける必要はない．送出された error はすべて `catch` に routing される．

### 型

`try` / `catch` 式の型は，`block` の成功値の型と，すべての `catch` arm 本体の型の共通型である．`match` と同様に，すべての分岐は同じ型を返さなければならない．

### exhaustiveness

`catch` の arm は，`block` 内で送出されうる error 型を網羅していなければならない (`match` の exhaustiveness と同じ)．wildcard arm を用いてよい．

```emela
try {
    read(path)
} catch {
    e -> defaultText()
}
```

### error channel の解消

`block` 内で送出される error を `catch` がすべてハンドルした場合，`try` / `catch` 式自体は error を送出しない．したがって `throws` を宣言しない関数でも，throwing な呼び出しを `try` / `catch` で包めば呼び出せる．

```emela
fn loadOrDefault(path: Path) -> Config uses { fs } {
    try {
        read(path)
        parse(path)
    } catch {
        e -> defaultConfig()
    }
}
```

`catch` arm の本体内で再度 `throw` すれば error を再送出できる．その場合，関数は対応する `throws` を宣言していなければならない．

### effect

`try` / `catch` の effect は `block` と到達しうる `catch` arm 本体の effect の和集合である．

```text
effects(try/catch) =
    effects(block)
    ∪ effects(reachable catch arm bodies)
```

## Panic

`panic` は組み込みの回復不能エラーである．

`panic` は現在の通常評価を終了し，プログラムを回復不能な異常終了へ移行させる．

`panic` は通常の値を返さず，型は `Never` であり，任意の期待型に適合する．

```emela
fn head<T>(xs: Array<T>) -> T uses {} {
    match Array.get(xs, 0) {
        Some(x) -> x
        None -> panic("empty array")
    }
}
```

`panic` は回復可能な失敗には使ってはならない．回復可能な失敗は `throws` または `Option` で表現する．

`panic` は `throws` でも `uses` でもなく，どちらの channel にも現れない．

WASM / WAMR では panic は runtime trap, runtime-provided panic handler に lowering しても良い．

WAMR embedder は panic message をログに出して異常終了しても良いが，通常評価へ復帰してはならない．

## エントリポイント

プログラムのエントリポイント `main` の `throws` は `Never` でなければならない (MUST)．

`throws Never` は non-throwing（error を送出しない）を意味し，`throws` 節の省略と等価である．したがって `main` は `throws` 節を省略するか，`throws Never` を宣言する．`Never` 以外の error 型を `main` の `throws` に宣言した場合はコンパイルエラーとする．

`main` の本体が throwing な呼び出しや `throw` を含む場合は，`main` 自身が `try` / `catch` で error を処理するか，`panic` で異常終了させなければならない．error を `main` の外へ伝播することはできない．

## IR Lowering

`throws E` を持つ関数は，IR レベルでは内部的に成功値と error の tagged union (`Ok` / `Err`) を返す関数へ lowering してよい．これは実装上の表現であり，表面言語には現れない．

```emela
throw e
```

```text
%err = make_internal_err e
return %err
```

```emela
let text = read(path)?
```

```text
%r = call @read(path)

switch_internal %r {
    Ok => block_ok
    Err => block_err
}

block_ok:
    %text = internal_payload %r 0
    continue %text

block_err:
    %e = internal_payload %r 0
    %out = make_internal_err %e
    return %out
```

```emela
try {
    read(path)
} catch {
    e -> defaultText()
}
```

```text
%r = call @read(path)

switch_internal %r {
    Ok => block_ok
    Err => block_catch
}

block_ok:
    %v = internal_payload %r 0
    continue %v

block_catch:
    %e = internal_payload %r 0
    %d = call @defaultText()
    continue %d
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

## Open Questions

- `throws A | B` のような union error 型を許すか．
- error 型の暗黙変換 (`From` / `Into`) を導入するか．
- `try` block 内から `catch` を飛び越えて関数の `throws` へ伝播する手段 (`?` の許可) を入れるか．
- `?` を使える型をユーザーが定義できる trait (例: `Unwrappable`) を導入するか．
