## 0017: Char and String Concatenation

Status: Draft

`Char` 型・文字リテラル・文字列連結，および `Int`/`Char`/`String` 間の最小変換を定義する仕様．

### Summary

- `Char` 型（Unicode scalar value，コードポイント）を追加する．
- 文字リテラル `'x'`（文字列と同じエスケープ）．
- 連結演算子 `++ : (String, String) -> String`（純粋）．
- 変換 `from_code(Int) -> Char` と `from_char(Char) -> String`（いずれも純粋）．

これらの変換・連結は純粋操作であり，capability effect を持たない（`uses {}`）．これだけで，数字 `d` を文字 `from_code(48 + d)` にして `from_char` で1文字の `String` にし，`++` で連結する，という純粋Emelaの `to_string` が書ける．

> **改訂（spec 0021）**: 当初これらの変換は `Char::from_code` / `String::from_char` という `::` 型パス構文の**言語ビルトイン**（コンパイラが専用ノードへ lower する）として定義された．spec 0021（Intrinsics）でこれらは通常の **intrinsic 関数** `char_from_code` / `string_from_char`（bare 名，stdlib の Core Prelude が宣言）へ移された．`++` の連結も同様に `string_concat` intrinsic が backing する．本節の残りの記述で `Char::from_code` / `String::from_char` とあるのは，現行では `char_from_code` / `string_from_char` と読み替える．`::` は enum variant 専用である（spec 0018 R7）．

### Specification

- **Char**: 1つの Unicode scalar value を表す primitive 型．コピー可能（heap 参照ではない）．型注釈で `Char` と書ける．
- **文字リテラル**: `'x'`．エスケープは文字列リテラルと同じ集合（`\n \t \\ \' \"`）．ちょうど1つの scalar value を表す．
- **`++`**: `String ++ String -> String`．純粋・結合的・左結合で，加算 `+` と同じ優先順位．新しい `String` を生成し，両被演算子は変更しない（`String` は immutable, 0007）．
- **`char_from_code(n: Int) -> Char`**（旧 `Char::from_code`）: コードポイント `n` の `Char`．`n` が有効な scalar value でない場合の扱いは実装定義（当面は下位ビットを採用してよい，Open Questions）．
- **`string_from_char(c: Char) -> String`**（旧 `String::from_char`）: `c` 1文字だけからなる `String`．
- これらはすべて純粋（`uses {}`，`throws` 無し）．platform 関数ではない（spec 0013 とは別物で，runtime 供給を要しない）．intrinsic として backend が native 命令にインラインする（spec 0021）．

### Examples

```emela
fn digit(d: Int) -> String {
  string_from_char(char_from_code(48 + d))
}

fn main() -> String {
  "ab" ++ digit(1) ++ digit(2)   -- "ab12"
}
```

### Compilation Notes

この節は非規範的である．

- `Char` は実行時にはコードポイントの整数（WebAssembly では `i32`，JavaScript では number）として表現する．
- JavaScript (Tier 2): `char_from_code` は恒等，`string_from_char` は `String.fromCodePoint(c)`，`++`（`string_concat`）は `a + b`．全コードポイントを正しく扱う．
- WebAssembly (Tier 1): `String` は線形メモリ上の `[len: i32][utf8 bytes]`（0013/0015）．`++`（`string_concat`）は `alloc(4 + len1 + len2)` ＋ 長さ書込み ＋ `memory.copy` ×2．`string_from_char` は **当面 ASCII（コードポイント < 128）の1バイトのみ**を符号化する（多バイト UTF-8 符号化は後続）．
- `Char` リテラルは typed IR (0012) の専用ノードとして保持する．変換・連結（`char_from_code` / `string_from_char` / `string_concat`）は，専用ノードではなく汎用の `IrExpr::Intrinsic { name, args, ret }` として保持し，backend が intrinsic 名で命令表を引く（spec 0021）．

### Open Questions

- 無効なコードポイント（サロゲート等）に対する `Char::from_code` の挙動（trap / クランプ / 下位ビット）．
- WebAssembly の `String::from_char` を多バイト UTF-8 に拡張する時期．
- `Char` の比較・`Char -> Int`（`Char.code`）など他の `Char` 演算の追加．
- 文字列の長さ・添字・部分文字列（0007 の Open Questions と共有）．
