# 0028: Generic Data Type Declarations

Status: Draft

ユーザー定義の **ジェネリック enum / record**（型パラメータ付きデータ型宣言）を定義する仕様．
0005（Enum）・0006（Records）に型パラメータを与え，0014（Generic Functions）の推論・単相化を
データ型へ拡張する．0014 の Open Questions が予告し，0005 の `enum Either<L, R>` 例を正式化する．

## Summary

- `enum Name<T, ...> { ... }` / `record Name<T, ...> { ... }` で型パラメータ付きデータ型を宣言できる．
- 型パラメータは variant の payload 型・フィールド型で使える．
- 名前付き型は自身を **payload / フィールドの位置で再帰的に参照してよい**（`List<T>` のような再帰型）．
- 構築時の型引数は payload / フィールドの実引数型から推論する．payload の無い variant（`Nil` 等）は
  期待型から決定する．
- 単相化（0014）を拡張する: 到達可能な `(型, 具体型引数)` の組ごとに単相版のレイアウトを生成し，
  typed IR（0012）に型変数は現れない．
- 型パラメータへの境界 (bounds) はデータ型宣言では付けられない（能力は使用側の関数・impl が要求する）．

## Motivation

組み込みの `Array<T>` / `Option<T>` はあるが，ユーザーは独自のジェネリックデータ型（`Either<L, R>`,
`Pair<A, B>`, `List<T>`, `Tree<T>`）を宣言できなかった．0005 は `enum Either<L, R>` を例示しながら
規範化しておらず，0020 は `impl<T: Show> Show for Array<T>` のようなパラメータ付き instance を定めた
のに，適用先のユーザー定義ジェネリック型が存在しなかった（層の逆転）．また stdlib の `List<T>`（0029）
は本仕様を前提とする．

## Specification

### 宣言

```emela
enum Either<L, R> {
    Left(L)
    Right(R)
}

record Pair<A, B> {
    first: A
    second: B
}
```

- 型パラメータは型名の直後の `< >` にカンマ区切りで並べる（0014 の関数と同じ構文規約: 空リスト不可，
  慣習として大文字始まり）．
- 宣言された型パラメータは，その宣言の variant payload 型・フィールド型の中で型として使える (MUST)．
  スコープは宣言に閉じる．
- 型パラメータに境界（`<T: Show>`）は付けられない (MUST NOT)．データ型は能力を要求しない．能力は
  それを使う関数（0020 の境界）や instance（パラメータ付き impl）が要求する．

### 再帰型

- 名前付き型（enum / record）は，自身（および相互に）を payload 型・フィールド型の位置で参照して
  よい (MUST)．

```emela
enum List<T> {
    Nil
    Cons(T, List<T>)
}
```

- これが有限のデータ表現を持つのは，enum payload / record が常にヒープ参照（0001）だからである．
  無限サイズの心配（値の直埋め込みによる再帰）は生じない．
- 再帰型の値も構築順により非巡回である（0024 の不変条件を破らない）．

### 型としての使用

- `Name<C1, ..., Cn>`（`C` は具体型または，ジェネリック文脈では型変数）を型の位置に書ける．
- 型引数の数は宣言と一致しなければならない (MUST)．部分適用（`Either<Int>`）はできない．

### 構築と型引数の決定

- **enum variant**: payload の実引数型から，0014 の構造対応付けで型引数を推論する．

```emela
let e = Left(1)              -- Either<Int, ?> ： R が決まらない → 期待型が必要
let e: Either<Int, String> = Left(1)    -- OK
```

- payload に現れない型パラメータ（上の `R`）や，payload の無い variant（`Nil`）は，**期待型**
  （束縛の注釈・引数位置・関数の戻り値型・match の他 arm 等）から決定する (MUST)．期待型からも
  決まらない場合はコンパイルエラーとする（0014 の「推論で一意に定まらなければエラー」と同じ規律）．
- **record**: フィールドの実引数型から推論する．record は全フィールドを与えるため，通常すべての
  型パラメータが決まる．

### match

- ジェネリック enum への `match` は，scrutinee の具体型引数で payload の型が定まる．exhaustiveness
  （0005）は従来どおり variant 集合に対して検査する．

```emela
fn from_either<L, R>(e: Either<L, R>, default: R) -> R {
    match e {
        Left(_) -> default
        Right(r) -> r
    }
}
```

### ジェネリック関数・trait との統合

- 0014 の型引数推論の構造対応付けに，`Name<D...>` ↔ `Name<A...>`（同名のユーザー定義型どうしを
  引数ごとに再帰的に対応付ける）を追加する (MUST)．`Array` / `Option` への規則の一般化である．
- 0020 のパラメータ付き instance（`impl<T: Show> Show for Pair<T, T>` 等）の対象型に，ユーザー定義
  ジェネリック型を使える．孤児規則（0020）は従来どおり適用される．

## Examples

```emela
record Pair<A, B> {
    first: A
    second: B
}

fn swap<A, B>(p: Pair<A, B>) -> Pair<B, A> {
    Pair { first: p.second, second: p.first }
}

fn main() -> Int {
    let p = Pair { first: 1, second: "one" }   -- Pair<Int, String>
    swap(p).second                             -- 1
}
```

## Compilation Notes

この節は非規範的である．

- **単相化**: 到達可能な `(型, 具体型引数)` の組ごとにレイアウト（enum は tag + payload 参照，record は
  フィールド参照列, 0001）を確定する．関数の単相化（0014）と同じ推移的展開で，typed IR（0012）には
  型変数が現れない．
- **表現の共有**: payload / フィールドは常に参照または primitive なので，多くの instantiation は同一の
  物理レイアウトを共有できる（`Pair<Int, Int>` と `Pair<Int, Bool>` は同型）．共有は最適化であり規範では
  ない．
- **再帰型**: 参照経由なので特別な扱いは不要．

## Open Questions

- variance（現状は不変 invariant のみ．subsumption 0023 と組み合わせた関数型フィールドの変性）．
- ジェネリック型の別名 (type alias)．
- 型パラメータの既定値・部分適用．
- 組み込み `Option<T>` / `Array<T>` を本仕様の機構上の宣言として再定義するか（0021 の「Int を stdlib
  型にする」拡張余地と連動）．
