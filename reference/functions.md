# Bindings & Functions

> **リファレンス** — 現在の規範。経緯・動機は末尾 [Provenance](#provenance) の RFC を参照。

束縛は不変で、反復は再帰で書く。関数は第一級の値であり、その型は
引数型・戻り値型に加えて [`throws`](errors.md) と [`uses`](effects.md) を持つ。

キーワード MUST / MUST NOT / SHOULD / MAY は RFC 2119 に従う。

## 用語

| 用語 | 定義 |
|---|---|
| **`let` 束縛** | 値に名前を与える。不変で再代入できない。 |
| **関数値** | 第一級の値としての関数。無名関数・クロージャを含む。 |
| **末尾位置** | 関数本体で、その式の値がそのまま関数の戻り値になる構文位置。 |

## 1. 束縛

- 変数はすべて **不変** である。再代入はできない。束縛には予約語 `let` を用いる。

```emela
let x = 1
let y = x + 1
```

## 2. 関数定義

```emela
fn add(x: Int, y: Int) -> Int {
    x + y
}
```

- `fn` で関数を定義する。本体は [block 式](expressions.md#1-block)である。
- 関数は **第一級の値** である。名前付き関数は関数値への束縛と解釈できる。
- 関数は lexical scope を持つ。
- トップレベル関数の再帰・相互再帰を許可する。
- 引数は **左から右へ** 評価する。`f(g(), h())` は `g()` → `h()` → `f(...)` の順に評価する。各式はちょうど一度だけ評価される。
- カリー化・部分適用は持たない。多引数関数の連鎖は [pipeline `|>`](expressions.md#6-pipeline-) で書く。

## 3. 関数値・クロージャ

関数は値であり、束縛・引数渡し・戻り値にできる。無名関数の構文は次の形。

```emela
fn (parameter_list) -> ReturnType uses EffectRow { block }
```

無名関数は外側の lexical scope の値を capture できる（クロージャ）。

```emela
fn make_adder(n: Int) -> ((Int) -> Int uses {}) uses {} {
    fn (x: Int) -> Int uses {} { x + n }   -- n を capture
}
```

関数を返す関数型は括弧で明確化する（`((Int) -> Int uses {}) uses {}`）。

## 4. 関数型

関数型は引数型・戻り値型・[`throws E`](errors.md#2-throws-と-throw)・[`uses F`](effects.md#2-関数型の-effect-row) からなる。

```text
(A, B) -> T throws E uses F
```

- `uses {}`（純粋）は省略してよい。`(Int) -> Int uses {}` と `(Int) -> Int` は等価である。
- `throws` / `uses` は型の一部であり、これらの異なる関数型は異なる型である。ただし effect row の[部分集合方向の適合（subsumption）](effects.md#6-subsumption-と-row-拡張)は許される。
- effect-row を型変数で受ける総称版は [effect-row 多相](effects.md#7-effect-row-多相)、型パラメータ付きの関数は [Generics](generics.md) が定める。

```emela
fn apply<T>(x: T, f: (T) -> T uses 'e) -> T uses 'e { f(x) }
```

## 5. 自己末尾呼び出し

Emela はループ構文を持たず、反復は再帰で書く。無限に回る処理（サーバーの accept ループ等）を書けるよう、**自己末尾呼び出しはスタックを消費しないことを保証する**。

- **保証（MUST）**: トップレベル関数 `f` の本体の **末尾位置** にある `f` 自身への直接呼び出し（ベア名 `f(...)`）は、スタックを消費してはならない。自己末尾再帰の列は深さによらず一定のスタックで実行される。
- これは最適化（任意）ではなく **保証（規範）** である。観測可能な意味・評価順・effect row・`throws` はすべて不変である。
- **末尾位置** は構文的に定める：関数本体 block の最終式／末尾位置にある block の最終式／末尾位置にある `if` の両 branch の最終式／末尾位置にある `match` の各 arm 本体の最終式／末尾位置にある `try`–`catch` の各 **catch arm** 本体の最終式。**`try` block の内部・`?` の対象・引数位置・演算子の被演算子・pipeline の左辺は末尾位置ではない**。
- **対象外**: 相互再帰（`f` → `g` → `f`）、関数値・関数リテラル経由の呼び出し、[trait メソッド](traits.md)経由の呼び出し、末尾位置にない自己呼び出し。これらの最適化は規定しない。

```emela
fn count_down(n: Int) -> Unit uses { Io } {
    if n == 0 {
        Io.print("done\n")
    } else {
        Io.print(n)
        count_down(n - 1)   -- 末尾位置：スタックを消費しない
    }
}

fn fact(n: Int) -> Int uses {} {
    if n == 0 { 1 } else { n * fact(n - 1) }   -- `*` の被演算子：末尾位置ではない（保証対象外）
}
```

## 未解決事項

- **相互再帰への拡張**（WASM tail-call 拡張の採用可否と連動）。
- 末尾自己呼び出しを表明・検査する属性（`@tailcall` 等）。
- ジェネリック関数・境界付き関数を **第一級の値** として扱う手段（現状は型引数が決まらないため不可、→ [Generics](generics.md)）。

## Provenance

- 束縛と不変性: [`../specs/0002`](../specs/0002-binding-and-mutation.md)
- 関数定義・呼び出し・評価順・第一級関数・クロージャ: [`../specs/0003`](../specs/0003-function-definitions-and-calls.md)
- 自己末尾呼び出しの保証: [`../specs/0045`](../specs/0045-self-tail-calls.md)
- 関数型の `throws` / `uses`: [`../specs/0008`](../specs/0008-function-types-with-effects.md)（→ [Effects](effects.md) / [Errors](errors.md)）
