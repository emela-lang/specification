## 0015: If Expression

Status: Draft

`if` を式として定義する仕様．

### Summary

`if cond { ... } else { ... }` は，`Bool` の条件で2つの branch を選ぶ**式**である．両 branch は同じ型を持ち，`if` 全体の型はその型になる．

### Motivation

Emela には条件分岐が無い（`match` は enum / `Option` 専用で，`Bool` や数値リテラルに対しては使えない）．符号やゼロの判定，再帰の終了など，多くの純粋なコード（例: `to_string`）が条件分岐を必要とする．

### Specification

```text
if cond { then_block } else { else_block }
```

- `cond` の型は `Bool` でなければならない (MUST)．
- `then_block` と `else_block` は block 式である (0004)．その値の型は一致しなければならない (MUST)．この共通型 `T` が `if` 式全体の型になる．
- `else` 節は必須である (MUST)．`else` の無い `if` は本仕様では認めない（Open Questions 参照）．
- `if` は式であり，束縛・引数・関数本体の末尾など，値が要求される位置に書ける．
- 評価は `cond` を評価し，`true` なら `then_block` のみ，`false` なら `else_block` のみを評価する（選ばれなかった branch は評価しない）．
- effect (0009): `if` 式の effect は `cond`・`then_block`・`else_block` の effect の和集合である．選ばれる branch は実行時に決まるため，型検査では両 branch の effect を合算する．
- 文的に使いたい場合は両 branch を `Unit` にする．

### Examples

```emela
fn classify(n: Int) -> Int {
  if n < 0 {
    0 - 1
  } else {
    if n == 0 { 0 } else { 1 }
  }
}
```

```emela
fn main() -> Unit uses { io } {
  if 0 < 1 {
    print("yes\n")
  } else {
    ()
  }
}
```

### Compilation Notes

この節は非規範的である．

- 短絡評価（選ばれた branch のみ評価）を保つよう lowering する．
- JavaScript: 条件演算子 `cond ? then : else`（branch が block の場合は即時関数等で表現してよい）．
- WebAssembly: `(if (result T) <cond> (then <then>) (else <else>))`．`Unit` は i32 の `0` で表現する．
- typed IR (0012) には専用の分岐ノードとして保持し，effect 注釈を残す．

### Open Questions

- `else` の無い `if`（両義性を避けるため当面は禁止）を `Unit` 用に許すか．
- `else if` を糖衣として明示的に許すか（現状は `else { if ... }` と書ける）．
