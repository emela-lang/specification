## 0017: Char and String Concatenation

Status: Draft

`Char` 型・文字リテラル・文字列連結，および `Int`/`Char`/`String` 間の最小変換を定義する仕様．

### Summary

- `Char` 型（Unicode scalar value，コードポイント）を追加する．
- 文字リテラル `'x'`（文字列と同じエスケープ）．
- 連結演算子 `++ : (String, String) -> String`（純粋）．
- ビルトイン変換 `Char::from_code(Int) -> Char` と `String::from_char(Char) -> String`（いずれも純粋）．

これらは言語ビルトイン（コンパイラが各 backend へ直接 lower する純粋操作）であり，capability effect を持たない（`uses {}`）．これだけで，数字 `d` を文字 `Char::from_code(48 + d)` にして `String::from_char` で1文字の `String` にし，`++` で連結する，という純粋Emelaの `to_string` が書ける．

### Specification

- **Char**: 1つの Unicode scalar value を表す primitive 型．コピー可能（heap 参照ではない）．型注釈で `Char` と書ける．
- **文字リテラル**: `'x'`．エスケープは文字列リテラルと同じ集合（`\n \t \\ \' \"`）．ちょうど1つの scalar value を表す．
- **`++`**: `String ++ String -> String`．純粋・結合的・左結合で，加算 `+` と同じ優先順位．新しい `String` を生成し，両被演算子は変更しない（`String` は immutable, 0007）．
- **`Char::from_code(n: Int) -> Char`**: コードポイント `n` の `Char`．`n` が有効な scalar value でない場合の扱いは実装定義（当面は下位ビットを採用してよい，Open Questions）．
- **`String::from_char(c: Char) -> String`**: `c` 1文字だけからなる `String`．
- これらはすべて純粋（`uses {}`，`throws` 無し）．platform 関数ではない（spec 0013 とは別物で，runtime 供給を要しない）．

### Examples

```emela
fn digit(d: Int) -> String {
  String::from_char(Char::from_code(48 + d))
}

fn main() -> String {
  "ab" ++ digit(1) ++ digit(2)   -- "ab12"
}
```

### Compilation Notes

この節は非規範的である．

- `Char` は実行時にはコードポイントの整数（WebAssembly では `i32`，JavaScript では number）として表現する．
- JavaScript (Tier 2): `Char::from_code` は恒等，`String::from_char` は `String.fromCodePoint(c)`，`++` は `a + b`．全コードポイントを正しく扱う．
- WebAssembly (Tier 1): `String` は線形メモリ上の `[len: i32][utf8 bytes]`（0013/0015）．`++` は `alloc(4 + len1 + len2)` ＋ 長さ書込み ＋ `memory.copy` ×2．`String::from_char` は **当面 ASCII（コードポイント < 128）の1バイトのみ**を符号化する（多バイト UTF-8 符号化は後続）．
- これらは typed IR (0012) では専用ノード（`Char` リテラル，`CharFromCode`，`StringFromChar`，`Concat`）として保持する．

### Open Questions

- 無効なコードポイント（サロゲート等）に対する `Char::from_code` の挙動（trap / クランプ / 下位ビット）．
- WebAssembly の `String::from_char` を多バイト UTF-8 に拡張する時期．
- `Char` の比較・`Char -> Int`（`Char.code`）など他の `Char` 演算の追加．
- 文字列の長さ・添字・部分文字列（0007 の Open Questions と共有）．
