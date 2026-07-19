# 0018: Qualified Imports and Calls

Status: Draft

import した関数を，bare 名だけでなく **修飾パス**（`int.to_string` / `std.int.to_string`）でも
呼べるようにする仕様．名前空間の衝突を，import を諦めずに使用箇所で解決できるようにする．
spec 0010（Modules, Imports, and Visibility）を拡張する．

> 注（spec 0036）：effect の操作は本仕様の唯一の例外で、suffix のベア名解決が働かず
> **修飾必須**である（`io.print` は可、ベア `print` は不可）．詳細は 0036 を参照．

## Summary

- `import a.b.f` は関数 `f` を，`f` で終わる**任意の非空 suffix**で参照可能にする．
  例: `import std.int.to_string` の後，`to_string(x)` / `int.to_string(x)` / `std.int.to_string(x)`
  はすべて同じ関数を呼ぶ．
- 同じ bare 名を束縛する複数の import を許可する（**import 時には拒否しない**）．衝突は
  使用箇所で解決する: 呼び出しパスが一意に解決できれば OK，複数に一致すれば曖昧エラー．
- これにより，`import std.int.to_string` と `import std.float.to_string` を併用しても，
  `int.to_string(x)` / `float.to_string(x)` と書き分ければ衝突しない．bare `to_string(x)` だけが
  曖昧エラーになる．

## Motivation

spec 0010 の import は，`import std.int.to_string` をすると bare 名 `to_string` だけを scope に
束縛する．複数モジュールが同名関数を export すると bare 名が衝突し，どちらか一方しか使えない
（後勝ち）．修飾呼び出しがあれば，import を分けたまま `int.to_string` と `float.to_string` で
区別でき，名前空間の衝突を避けられる．

## Specification

呼び出し対象・参照対象は，1つ以上の識別子をドットで繋いだ**パス** `p1.….pk`（k ≥ 1）で
書ける．パス解決は次の規範規則に従う．

- **R1（パス）**: 参照式・呼び出しの callee は，ドットパス `p1.….pk`（k ≥ 1）であってよい．
  k = 1 は従来の bare 名と同じ．
- **R2（suffix 束縛）**: `import a1.….an.f`（最後の要素 `f` が関数名）は，`f` をその
  **full path `[a1,…,an,f]` の，`f` で終わる全ての非空 suffix**で参照可能にする．
  すなわち `f`，`an.f`，…，`a1.….an.f` のいずれでも呼べる．
- **R3（suffix 一致）**: 呼び出しパス `[q1,…,qm]` は，import された関数 `F` の full path の
  suffix と一致するとき `F` の候補になる．
- **R4（衝突は import 時に拒否しない）**: 同じ bare 名を束縛する複数の import を，import 時に
  エラーにしてはならない（MUST NOT）．衝突の検出は使用箇所まで遅延する．
- **R5（一意解決 / 曖昧）**: あるパスの候補が
  - ちょうど 1 個 → その関数に解決する．
  - 2 個以上 → コンパイルエラー（曖昧）．エラーは候補それぞれの full path を列挙する（MUST）．
  - 0 個 → 名前未定義．
- **R6（優先順位）**: ドットパス `p1.….pk` の解決は次の優先順位で行う:
  1. ローカル束縛（`let` / 引数）— bare 名のみ．
  2. 同一コンパイル単位（entry）の関数 — bare 名．import より優先（shadow）する．
  3. import された関数の suffix 一致（R3）．
- bare 名（k = 1）は従来通り常に使える（後方互換）: full path の最短 suffix `[f]` に当たる．
- **R7（型パスは `::`，`.` とは別系統）**: 型に紐づく名前 — enum variant（`Enum::Variant`，spec 0005）
  — は，ドット `.` ではなく **二重コロン `::`** で書く型パスである．型パスは本節の `.` パス解決
  （R1〜R6）とは**構文的に分離**され，互いに衝突しない．`::` の左辺は enum 型名であり，
  `Enum.Variant` は error とする．
  - かつて `::` で特別扱いしていた変換 `Char::from_code` / `String::from_char`（spec 0017）は，
    spec 0021 で通常の intrinsic 関数 `char_from_code` / `string_from_char`（bare 名）へ移された．
    したがって `::` は enum variant 専用であり，`Char::…` / `String::…` / `Array::…` の型パスは
    もはや解決しない．
- private（`pub` でない）関数は，importer から**修飾パスでは参照できない**（bare 名での
  既存の振る舞いは spec 0010 / 実装に従う．本仕様はこれを変更しない）．

## Examples

```emela
-- int.emel:    module int;  pub fn to_string(n: Int) -> String { ... }
-- float.emel:  module float; pub fn to_string(x: Float) -> String { ... }

import std.int.to_string
import std.float.to_string   -- R4: import 時に拒否しない

fn main() -> Unit uses { io } {
  -- to_string(1)            -- R5: 曖昧（int と float の両方に一致）→ コンパイルエラー
  int.to_string(1)           -- OK（一意）
  std.int.to_string(1)       -- OK（full path も一意）
  float.to_string(2.0)       -- OK（一意）
}
```

```emela
import geometry.square

fn main() -> Int {
  square(5) + geometry.square(3)   -- bare も修飾も同じ関数
}
```

## Compilation Notes

この節は非規範的である．

- 各関数は，bare 名が全関数で一意なら bare 名のまま，衝突する import 関数だけ full path を
  identifier-safe にマングルした名前（例: `std.int.to_string` → `std__int__to_string`）を backend
  シンボルに用いる．一意な import（例: `std.io.print`）は bare 名のまま emit される．
- generic 関数（spec 0014）のマングル（`identity__Int`）とは別系統に保つ．
- typed IR（spec 0012）・各 backend は関数を一意なシンボル名で扱うので，同名関数が共存しても
  衝突しない．パス解決は型検査時に確定し，lowering は解決済みのシンボルを emit する．

## Open Questions

- 同一 entry の関数と import が同じ bare 名のとき: ローカル shadow（本仕様の R6-2）で確定とするか，
  曖昧エラーにするか．
- 推移的 import（モジュールが別モジュールを import する）における修飾子の付与規則．
- モジュール全体のマージと private helper の可視性（bare 名での既存の漏れを含む）の整理．
- 修飾された一次関数値（`let f = int.to_string`）の扱い．
- ~~enum 名とモジュール名が同一（2 セグメントで衝突）のときの解決．~~
  解決済み: enum variant は `::`（型パス），モジュール修飾呼び出しは `.` と構文的に分離した（R7）．
  よって `Enum::Variant` と `module.function` は衝突しない．
