## 0016: Integer Division and Remainder

Status: Draft

整数の除算 `/` と剰余 `%` を定義する仕様．

### Summary

`/`（除算）と `%`（剰余）を二項演算子として追加する．`Int / Int -> Int`（0方向への切り捨て）と `Float / Float -> Float`（実数除算），`Int % Int -> Int`（被除数の符号を持つ剰余）．0 による整数除算・剰余は trap（回復不能）．

### Motivation

算術は `+ - *` しか無く，桁の取り出し（`n / 10`，`n % 10`）ができない．純粋Emelaで数値→文字列変換などを書くために除算・剰余が要る．

### Specification

- `/` と `%` は乗算 `*` と同じ優先順位の左結合二項演算子である．
- 型:
  - `/` は両辺が一致する数値型に適用する．`Int / Int -> Int`，`Float / Float -> Float` (MUST)．
  - `%` は `Int % Int -> Int` のみ (MUST)．Float への `%` は型エラー．
- `Int / Int` は**0方向への切り捨て**（truncated division）．例: `7 / 2 == 3`，`-7 / 2 == -3`．
- `Int % Int` は被除数の符号を持つ剰余で，`a == (a / b) * b + (a % b)` を満たす．例: `7 % 2 == 1`，`-7 % 2 == -1`．
- **0 による整数の除算・剰余は trap である**（回復不能，spec 0011 の panic と同様に通常評価へ復帰しない）．trap は `uses`（capability）にも `throws`（error channel）にも現れない．
- `Float / 0.0` は IEEE-754 に従い（`inf`/`nan`），trap しない．

### Examples

```emela
fn last_digit(n: Int) -> Int {
  n % 10
}

fn halve(n: Int) -> Int {
  n / 2
}
```

### Compilation Notes

この節は非規範的である．

- WebAssembly: `Int` は `i32.div_s` / `i32.rem_s`，`Float` は `f64.div`．`i32.div_s`/`i32.rem_s` は 0 除数で trap するため，整数 0 除算 trap は追加コードなしで満たされる．
- JavaScript (Tier 2): `Int` 除算は `(a / b) | 0`，剰余は `a % b`，`Float` は `a / b`．現状の JS backend は 0 除算で trap せず（`(a/0)|0 === 0`），この点は WebAssembly と挙動が異なる既知の差分である．将来 0 チェックのガードで揃える．

### Open Questions

- 0 除算を trap ではなく `throws`（回復可能 error）にする選択肢．
- ビット演算やシフトなど他の整数演算をいつ追加するか．
