# 0000: Minimal Core Language

Status: Draft

## Summary

このドラフトは、Emela の最初の実行可能な部分集合を定義します。対象は、`fn` による
関数定義、ブロック式、ローカル束縛、プリミティブ値、明示的なプログラムエントリを
持つ小さな式言語です。

モジュール、代数的データ型、エフェクト、標準ライブラリを追加する前に、Native と
WebAssembly の両方へコンパイルできる最小の有用なコンパイラ対象を作ることを目的
とします。

## Motivation

最初の言語層では、関数型プログラムをコンパイルして実行するために必要な問いだけに
答えます。

- プログラムとは何か。
- 式とは何か。
- 関数をどう表すか。
- 副作用を伴う関数をどう表明するか。
- どのプリミティブ値が存在するか。
- Native/WASM 間で同一でなければならない挙動は何か。

この層を意図的に小さく保つことで、後続仕様を評価しやすくします。

## Specification

### Program

Emela プログラムはトップレベル関数定義の集合です。

実行可能なプログラムには、`main` という名前のトップレベル関数がちょうど 1 つ
MUST 存在します。

```emela
fn main() {
}
```

`main` は 0 個の引数を取ります。戻り値の型と実行可能ファイルの終了挙動は、この
ドラフトでは固定しません。

### Lexical Form

ソーステキストは UTF-8 です。

識別子は ASCII 英字または `_` で始まり、後続に ASCII 英字、数字、`_` を取れます。

関数名は、識別子の末尾に `!` を 1 つ付けることができます。末尾 `!` は、その関数が
副作用を伴う可能性を表明する effect marker です。

```emela
fn print!(value) {
  ()
}
```

`!` は関数名の末尾でのみ使用できます。ローカル変数名、型名、途中に `!` を含む名前は
このドラフトでは無効です。

整数リテラルはデフォルトで 10 進数です。

空白はトークンを分離します。隣接するトークンを分離する必要がある場合を除き、
意味を持ちません。

改行は、ブロック内の item を分離できます。ブロック外では、改行は通常の空白として
扱います。

行コメントは `--` で始まり、行末まで続きます。

### Primitive Types

初期コア言語は、次のプリミティブ型を定義します。

| Type | 意味 |
| --- | --- |
| `I32` | 符号付き 32-bit 整数 |
| `Bool` | `true` または `false` の真偽値 |
| `Unit` | `()` と書く単一の値 |

`I32` 算術は two's-complement の wrapping semantics を MUST 使用します。これは WASM
の整数演算と一致し、ターゲット依存のオーバーフロー挙動を避けるためです。

### Expressions

初期のプログラム構文と式文法は次の通りです。

```text
program = function*

function =
  attribute* fn function_name ( parameter_list? ) block

attribute =
  #[ attribute_name ( attribute_argument_list? ) ]

attribute_name =
  identifier

attribute_argument_list =
  identifier (, identifier)*

function_name =
  identifier "!"
  | identifier

parameter_list =
  identifier (, identifier)*

block =
  { block_item* }

block_item =
  binding
  | expr

binding =
  identifier = expr

expr =
  integer
  | true
  | false
  | ()
  | identifier
  | function_name ( argument_list? )
  | expr . identifier ( argument_list? )
  | expr binary_operator expr
  | if expr then expr else expr
  | block
  | ( expr )

argument_list =
  expr (, expr)*

binary_operator =
  +
  | -
  | *
  | ==
  | <
```

関数呼び出しは `name(arg1, arg2)` の形で書きます。

属性は関数定義の直前に書きます。属性の意味は個別仕様で定義します。初期コア言語では、
platform capability を表す `#[requires(...)]` を
[0003: Platform Capabilities](0003-platform-capabilities.md) で定義します。

メソッド呼び出しは `receiver.method(arg1, arg2)` の形で書きます。メソッド探索、
trait 制約、演算子 desugaring は [0002: Traits and Operators](0002-traits-and-operators.md)
で定義します。

ブロックは式です。ブロックは 0 個以上の item を上から順に評価します。

item が束縛の場合、その名前を後続 item のスコープに導入します。item が式の場合、
その式を評価します。

ブロック全体の値は、最後の式 item の値です。式 item が存在しない空ブロック、または
束縛だけを含むブロックは `()` に評価されます。

### Functions

関数は `fn` で定義します。

```emela
fn add(x, y) {
  x + y
}

fn main() {
  add(20, 22)
}
```

関数は複数の仮引数を取れます。関数値、ラムダ式、currying は後続仕様で導入します。

末尾に `!` を持つ関数は、エラー、I/O、状態変更などの effect を発生させる可能性を
表明します。

```emela
fn read_line!() {
  ()
}
```

`!` は effect system への表層構文上の入口です。詳細は
[0001: Effect System](0001-effect-system.md) で定義します。

### Bindings

関数ブロック内では、`name = expr` が immutable なローカル束縛を導入します。

```emela
x = 40
x + 2
```

束縛された名前は、その束縛より後にある同じブロック内の item でスコープに入ります。

同じブロック内で同じ名前を再束縛することは、このドラフトでは禁止します。これは
mutation ではなく immutable binding であり、`x = 42` は代入文ではありません。

トップレベルの `name = expr` は無効です。トップレベルには `fn` による関数定義だけを
置けます。

### Conditionals

`if` の条件式は `Bool` 型を MUST 持ちます。

両分岐は同じ型を MUST 持ちます。

```emela
if true then 1 else 0
```

### Operators

初期コア言語は、次の二項演算子構文を予約します。

| Operator | Trait method | 意味 |
| --- | --- | --- |
| `+` | `Add.add` | 加算 |
| `-` | `Sub.sub` | 減算 |
| `*` | `Mul.mul` | 乗算 |
| `==` | `Eq.eq` | 等価性 |
| `<` | `Ord.lt` | less-than |

二項演算子は組み込み関数ではなく、trait method call への糖衣構文です。

```emela
x + y
```

上の式は概念的に次の式へ desugar されます。

```emela
x.add(y)
```

`+` は、左辺の型が `Add` trait を実装している場合にのみ使用できます。その他の
演算子も対応する trait を実装している型に対してのみ使用できます。

`I32` は初期環境で `Add`、`Sub`、`Mul`、`Eq`、`Ord` を実装します。`I32` の
`Add`、`Sub`、`Mul` は two's-complement の wrapping semantics を MUST 使用します。

演算子の優先順位は、このドラフトでは意図的に未定義とします。優先順位が仕様化
されるまで、例では括弧を SHOULD 使用します。

### Effects

副作用とエラーは effect system で扱います。

初期コア言語では、effect の詳細な型付け規則は定義しません。ただし、関数名の末尾
`!` は、その関数の呼び出しが pure ではない可能性を表明する予約済み構文です。

`!` を持たない関数は pure とみなされます。pure な関数は、未処理の effect を持つ
関数を直接呼び出してはなりません。正確な effect propagation と effect handling は
[0001: Effect System](0001-effect-system.md) で定義します。

### Evaluation

評価は strict です。関数引数は関数本体より先に評価されます。

トップレベル定義は immutable です。

コア言語には、暗黙の mutation、暗黙の I/O、例外、停止性保証はありません。
エラーや副作用は effect system によって明示的に扱います。

### Type Checking

初期型システムは静的に検査されます。

型を割り当てられないプログラムは、コード生成前に MUST reject されます。

このドラフトでは型注釈を要求しません。後続仕様で、型注釈の構文と正確な推論
アルゴリズムを追加できます。

## Examples

```emela
fn main() {
}
```

```emela
fn main() {
  x = 40
  x + 2
}
```

```emela
fn is_zero(x) {
  x == 0
}

fn main() {
  if is_zero(0) then 1 else 0
}
```

```emela
fn add(x, y) {
  x + y
}

fn main() {
  add(20, 22)
}
```

## Compilation Notes

`I32` は WASM の `i32` に直接対応します。

Native ターゲットでは、`I32` は明示的な wrapping 挙動を持つ固定幅の符号付き
32-bit 整数演算へ lower するべきです。

このドラフトでは関数値とクロージャを定義しません。後続仕様で関数を first-class
value として導入する場合、closure conversion などの実装戦略を別途定義します。

`!` 付き関数は、Native では明示的な runtime capability、WASM では import または
host binding に lower される可能性があります。effect の lower 方針は
[0001: Effect System](0001-effect-system.md) で扱います。

## Open Questions

- 関数定義は最初のバージョンから明示的な型注釈を持つべきか。
- `main` の戻り値型を固定するべきか。それとも実行可能ファイルの終了挙動を通常の
  式評価から分離するべきか。
- 再帰は minimal core に含めるべきか。それともトップレベル再帰の別仕様で導入する
  べきか。
- `!` は effect を持つ可能性の表明に留めるか、effectful であることの証明まで要求するか。
- effect marker を関数名の一部として扱うか、名前とは別の属性として扱うか。
- ソースファイルの拡張子は何にするか。
