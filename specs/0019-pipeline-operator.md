# 0019: Pipeline Operator

Status: Draft

左辺の値を右辺の関数呼び出しの第一引数に流し込む二項演算子 `|>` を定義する仕様．
ネストした呼び出しを，データの流れる順（左→右）で書けるようにする．

## Summary

- 二項演算子 `|>`（pipe）を追加する．`lhs |> f(a, b)` は `f(lhs, a, b)` と等価（**第一引数挿入**）．
- 右辺が呼び出しでない関数値式のとき，`lhs |> f` は `f(lhs)`．
- **最低優先順位・左結合**．`a |> f |> g` は `g(f(a))`．
- 純粋な構文糖であり，型・effect・throws・実行時の意味はすべて脱糖後の呼び出しと一致する．新しい意味論も実行時表現も導入しない．

## Motivation

Emela はネストした呼び出し `c(b(a(x)))` を内側から外側へ読む必要があり，変換の連鎖が読みにくい．`x |> a |> b |> c` と書ければ，データの流れる順に上から読める．

Emela はカリー化・部分適用を持たない（spec 0003）ため，`x |> f` を単なる `f(x)` に限ると多引数関数を連鎖できない．そこで多引数呼び出しをそのまま繋げられる「第一引数挿入」方式を採る．

## Specification

`|>` は二項演算子であり，次の規則で**脱糖（純粋な構文変換）**される．`|>` 式の意味は脱糖後の式で定義する．

- **P1（優先順位・結合性）**: `|>` は全二項演算子の中で**最低優先順位**かつ**左結合**である (MUST)．
  - 算術・比較より低い: `a + b |> f` は `f(a + b)`．
  - 左結合: `a |> f |> g` は `(a |> f) |> g`，すなわち `g(f(a))`．
- **P2（第一引数挿入）**: 右辺が呼び出し式 `f(a1, …, an)`（n ≥ 0）のとき，`lhs |> f(a1, …, an)` は `f(lhs, a1, …, an)` と等価である (MUST)．callee `f` は任意の式でよい．
- **P3（bare 右辺）**: 右辺が呼び出し式でない式 `e` のとき，`lhs |> e` は `e(lhs)` と等価である (MUST)．`e` は関数値に評価されなければならない（さもなくば型エラー）．
- **P4（末尾 `?`）**: 右辺末尾の error 伝播演算子 `?`（spec 0011）は，挿入の**後**に適用する．`lhs |> g?` は `(lhs |> g)?` と等価である (MUST)．これにより `data |> validate? |> transform?` は `transform(validate(data)?)?` と書ける．
- **P5（評価順）**: 脱糖後の呼び出しの評価順（spec 0003）に従う．`lhs |> f(a, b)` は `lhs`，`a`，`b` の順に評価してから `f` を呼ぶ．左辺は常に最初に評価され，ちょうど一度だけ評価される．
- **型・effect・throws**: `|>` 式の型・effect row・throws 節は，脱糖後の呼び出しのそれに一致する (MUST)．`|>` 自体は新たな effect も throws も導入しない．generic 関数（spec 0014）への pipe も，脱糖後の呼び出しと同様に型引数を推論する．

## Examples

```emela
fn add1(x: Int) -> Int { x + 1 }
fn double(x: Int) -> Int { x * 2 }
fn scale(x: Int, factor: Int) -> Int { x * factor }

fn main() -> Int {
  21 |> add1 |> double |> scale(1)   -- scale(double(add1(21)), 1) == 44
}
```

第一引数挿入（多引数関数の連鎖）:

```emela
xs |> map(double) |> filter(is_even) |> fold(0, add)
-- fold(filter(map(xs, double), is_even), 0, add)
```

error 伝播（spec 0011）との組み合わせ:

```emela
fn run(s: String) -> Int throws ParseError {
  s |> parse? |> validate?      -- validate(parse(s)?)?
}
```

## Compilation Notes

この節は非規範的である．

- `|>` は **frontend（parser）で脱糖**され，typed IR（spec 0012）や各 backend には専用ノードを持たない．型検査・lowering・コード生成は通常の呼び出し（`Call`）だけを見る．
- したがって本仕様の追加で，IR・JavaScript backend・WebAssembly backend はいずれも変更を要しない．
- 脱糖は構文的変換なので，左辺は引数位置へ移動されるだけで複製されない（P5 の「一度だけ評価」を自然に満たす）．

## Open Questions

- 修飾パス右辺（`x |> int.to_string`，spec 0018 の修飾呼び出し）の解決規則．当面は bare 名・呼び出し・関数値式を主対象とする．
- enum variant constructor を右辺に取る pipe（`x |> Some`）の可否と意味．
- 第一引数以外へ挿入するためのプレースホルダ構文（例: `x |> f(a, _)`）を将来追加するか．
- `|>` と単独の `|`（将来のビット演算・パターン区切りなど）の字句的な棲み分け．
