## 0051: Bytes — an Immutable Byte Sequence

Status: Draft

不変のバイト列型 **`Bytes`** を導入する仕様。`String`（0007：UTF-8 の text、要素は Unicode scalar）に対し、`Bytes` は**解釈を持たない生バイトの列**（要素は `0–255` の byte、byte 単位で数える）である。ランタイム表現は `String` と同一（`[len][bytes]`）で、違いは「scalar 単位で数え UTF-8 を強制するか（String）／byte 単位でそのまま扱うか（Bytes）」だけである。`Socket`（0050）・HTTP（0044/0046 改訂）が生バイトを扱うために要る。0044 の Open Question「`Bytes` 型の導入」を回収する。

### Summary

- `Bytes` は immutable なバイト列である。要素は byte（`Int` で 0–255 を表す）、長さ・添字はすべて **byte 単位**である（`String` の scalar 単位と対比）。
- 基本操作は 0021 の **intrinsic**（bare 名、Core Prelude が宣言し backend が native へ lower）＋ stdlib（`std.bytes`）の安全ラッパで供給する。`String` / `Array` と同じ形（0007/0030）。
- `String` ⇄ `Bytes` の相互変換を与える：`to_bytes(String) -> Bytes`（UTF-8 encode、全域）／`string_from_bytes(Bytes) -> Option<String>`（UTF-8 decode、不正なら `None`。panic しない、0011）。
- 演算子 `++`（`Concat`、0017/0020）と `==`（`Eq`、0020）を `Bytes` に実装する（`impl Concat for Bytes` / `impl Eq for Bytes`）。連結・等価が既存の演算子でそのまま書ける。
- `Bytes` は組み込み型（0001 と同格、型名は prelude で参照可能）。操作は `std.bytes` から import する（`String` の操作が `std.string` にあるのと同じ）。
- リテラル構文（`b"..."` 等）は本仕様では導入せず、`to_bytes("...")` で構築する（Open Questions）。

### Motivation

`String` は「text（UTF-8、scalar 単位）」として設計され（0007）、任意のバイト列を安全に保持できない。しかし `Socket`（0050）の read/write、HTTP のリクエスト／レスポンス body（0044/0046）は**任意のバイト列**を扱う必要がある。0044 は body を暫定的に UTF-8 `String` に制限し、非 UTF-8 を `NonUtf8Body` エラーにしていたが、これは表現力の欠落であり Open Question に挙げられていた。

`Bytes` を導入すると、生ソケットの上に HTTP を Emela の handler（0049/0050）として書ける：`Socket.read` が返す `Bytes` をパースして `Request` を作り、`Response` を `Bytes` にシリアライズして `Socket.write` する。text が要る箇所（ヘッダ名・URL 等、ASCII）は `string_from_bytes` で境界を越える。

`String` と表現を共有するため、backend・memory model（0024）・ARC（0048）への追加コストは最小である。

### Specification

以下、キーワードは RFC 2119 に従う。

#### 型と表現

- **T1（型）**: `Bytes` は組み込み型である（0001）。型名は Core Prelude で参照可能とし、`import` を要さない（`String` と同じ）。値は immutable である（MUST）。
- **T2（要素と単位）**: `Bytes` の要素は byte であり、`Int`（0–255）で表す（MUST）。長さ・添字・部分列はすべて **byte 単位**で数える（`String` の scalar 単位と異なる、MUST）。
- **T3（表現）**: ランタイム表現は `String` と同一の `[len: i32][raw bytes]` とする（0024）。`String` との違いは意味論（byte 単位・無解釈）のみであり、backend の値表現・ARC ヘッダ（0048）は `String` と共有してよい（MAY）。

#### 操作（intrinsic ＋ stdlib）

生アクセサは 0021 の intrinsic、安全版は `std.bytes` の `pub fn` ラッパとする（0007 の `array_get_unchecked` / `array_get`、0030 の string と同型）。

```
-- intrinsic（Core Prelude が宣言、backend が native へ lower）
intrinsic fn bytes_length(b: Bytes) -> Int uses {}
intrinsic fn bytes_get_unchecked(b: Bytes, i: Int) -> Int uses {}   -- 0 <= i < length 前提、byte 値 0–255
intrinsic fn bytes_slice(b: Bytes, start: Int, end: Int) -> Bytes uses {}  -- 半開区間 [start, end)
intrinsic fn bytes_concat(a: Bytes, b: Bytes) -> Bytes uses {}
intrinsic fn bytes_from_string(s: String) -> Bytes uses {}          -- UTF-8 encode（全域）
```

- **B1（length）**: `bytes_length(b)` は byte 数を返す（MUST）。
- **B2（unchecked get）**: `bytes_get_unchecked(b, i)` は前提 `0 <= i < bytes_length(b)` の下で `i` 番目の byte を `0–255` の `Int` として返す（未検査、0007 と同型）。
- **B3（safe get）**: `std.bytes` の `pub fn byte_at(b: Bytes, i: Int) -> Option<Int>` は境界外で `None` を返し panic しない（0011。`std.string.char_at` と同型、MUST）。
- **B4（slice）**: `bytes_slice(b, start, end)` は byte 単位の半開区間 `[start, end)` の部分列を新たに返す（元の `b` は不変）。範囲は `0 <= start <= end <= length` にクランプする（実装定義でなく、`std.string` の slice 規則に合わせる）。
- **B5（concat / `++`）**: `bytes_concat(a, b)` は連結した新しい `Bytes` を返す（純粋コピー、元は不変）。`impl Concat for Bytes`（0020）が `bytes_concat` を本体に持ち、`a ++ b` が `Bytes` で使える（MUST）。
- **B6（`String` → `Bytes`）**: `bytes_from_string(s)`（stdlib 公開名 `to_bytes(s: String) -> Bytes`）は `s` の UTF-8 バイト列を返す。全域である（MUST）。
- **B7（`Bytes` → `String`）**: `std.bytes` の `pub fn string_from_bytes(b: Bytes) -> Option<String>` は、`b` が妥当な UTF-8 なら `Some(String)`、そうでなければ `None` を返す（MUST。panic しない、0011）。実体は UTF-8 妥当性を検査する intrinsic に依る（Compilation Notes）。
- **B8（`Array<Int>` ⇄ `Bytes`、MAY）**: 利便のため `bytes_from_array(a: Array<Int>) -> Bytes`（各要素を mod 256 で byte 化、全域）と `bytes_to_array(b: Bytes) -> Array<Int>`（各 byte を `Int` 0–255 に）を `std.bytes` に置いてよい（MAY）。
- **B9（等価 / `==`）**: `impl Eq for Bytes`（0020）は byte 列の逐次比較で等価を定める（MUST）。長さと各 byte が一致するときのみ等しい。

#### 空と構築

- **C1**: 空の `Bytes` は `to_bytes("")` で得る。専用リテラルは導入しない（Open Questions）。

### Examples

HTTP レスポンス行を `Bytes` で組み立て、`Socket` に書く（0050 と接続）：

```
import std.socket
import std.bytes

fn write_status_line(conn: Connection) -> Unit uses { Socket } {
    Socket.write(conn, to_bytes("HTTP/1.1 200 OK\r\n\r\n"))
}
```

リクエストの先頭行を text として読む（byte を UTF-8 として解釈、非 UTF-8 は `None`）：

```
import std.bytes

fn head_as_text(raw: Bytes) -> Option<String> uses {} {
    string_from_bytes(bytes_slice(raw, 0, 16))
}
```

byte 単位の length / 添字（scalar 単位の `String` と対比）：

```
let b = to_bytes("héllo")     -- "é" は UTF-8 で 2 byte
-- bytes_length(b) == 6        （byte 単位）
-- "héllo" の String length は 5（scalar 単位、0007）
```

### Compilation Notes

この節は非規範的な実装上の補足である。

- **`String` との表現共有**: `Bytes` は `String` と同じ `[len][bytes]` 表現（0024）を用いる。`bytes_length` / `bytes_get_unchecked`（byte index）は `String` の byte 長・byte アクセスと同じ命令列に lower でき、`string_concat`（0017）と同じ連結ルーチンを `bytes_concat` に再利用できる。ARC（0048）のヘッダ・drop も `String` と共有する。差分は「`String` は scalar 単位で数える上位 API を持つ／`Bytes` は byte 単位」だけである。
- **intrinsic 対応**: `crates/emela-backend-wasm/src/lib.rs` の intrinsic 群（`string_length` / `string_concat` 等）に `bytes_*` を追加する（`wasm_provides_intrinsic` / `intrinsic_wasm` 相当）。`bytes_from_string` は表現共有ゆえ identity（同じバイト列を `Bytes` として再解釈）で済む。
- **UTF-8 decode（B7）**: `string_from_bytes` は UTF-8 妥当性検査を要するため、単純な identity ではない。妥当性を検査して `Option<String>` を返す intrinsic（またはそれを包む stdlib）で供給する。
- **JS backend**: `Bytes` は `Uint8Array`、`String`⇄`Bytes` は `TextEncoder`/`TextDecoder`（decode は fatal でなく妥当性判定して `Option`）で実装できる。

### Open Questions

- **リテラル構文**: `b"..."`（byte string literal、エスケープ）を導入するか。当面 `to_bytes("...")` で代替する。
- **`Show` / 16進表現**: `Bytes` の `to_string`（hex ダンプ等）を標準化するか。
- **可変ビルダ**: 逐次追記のための効率的な builder（現状は `++` の純粋コピー）。0007 の「可変長配列は別定義」と同じ論点。
- **`String` の byte ビュー**: `String` から UTF-8 バイトを取り出す `to_bytes` の逆（`Bytes` の scalar ビュー）を持つか。現状は `string_from_bytes` の一方向。
- **`Array<U8>` との関係**: 将来 `U8` 型を導入した場合の `Bytes` と `Array<U8>` の関係（別型のまま／別名）。

### 他仕様との関係

- **0001 / 0007（Core Values / Arrays and Strings）**: `Bytes` を組み込み型として追加する。`String` は text（scalar 単位）、`Bytes` は生バイト（byte 単位）と役割を分ける。表現は共有する。
- **0017 / 0020（Concat / Traits）**: `impl Concat for Bytes`（`++`）・`impl Eq for Bytes`（`==`）を与える。演算子 trait 機構をそのまま使う。
- **0011（Error handling）**: `byte_at` / `string_from_bytes` は不在・不正を `Option` で返し panic しない。
- **0021（Intrinsics）**: `bytes_*` を intrinsic として追加する。
- **0024 / 0048（Memory Model / ARC）**: `String` と表現・RC・drop を共有する。
- **0030（Stdlib String）**: `std.bytes` を `std.string` と同型に整備する。
- **0050（Socket）**: `Socket.read`/`write` の境界型を `Bytes` に確定する（0050 B1 の依存を解消）。
- **0044（HTTP Client）**: body を任意 `Bytes` で扱える基盤を与える（0044 の `Bytes` Open Question を回収）。text が要る箇所は `string_from_bytes`。
