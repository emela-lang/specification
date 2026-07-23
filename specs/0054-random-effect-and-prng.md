# 0054: Random — the `Random` Effect and a Seedable PRNG

Status: Draft

乱数を 2 つの層で提供する仕様。**Part A** は WASI 0.2 `wasi:random/random@0.2.0` にバインドしたプリミティブ effect **`Random`**（OS エントロピー源、暗号論的に安全・**非**シード可能）。**Part B** は決定論的で再現可能な**シード可能 PRNG**（xorshift32）を、effect ではない**純粋な値と関数**として提供する。両者は `std.random` モジュールに同居する。0049 の意味で `Random` はプリミティブ effect であり、PRNG は capability を持たない純粋計算である。

## Summary

- 組み込み capability に **`Random`** を追加する（0009 の拡張。`Io` / `Clock` / `Socket` と並列の独立 primitive）。**infallible**（乱数取得は失敗しない。`Clock` と同じく `throws` を持たない）。
- platform 関数は最小の 2 つ：`random.raw_int`（一様な `Int`）と `random.raw_bytes`（暗号論的に安全なバイト列）。lower 先は component backend の `wasi:random/random`（WASI 0.2）。
- 公開 API（`std.random`）は effect 操作 `Random.int()` / `Random.bytes(len)`。
- **シード可能 PRNG** を、同じ `std.random` に純粋な値として提供する：`record Prng`、`seed(s)`、`next(g)`。決定論的・再現可能で capability を要求しない（`uses {}`）。アルゴリズムは xorshift32（0053 のビット演算を使う）。
- エントロピー種は `seed(Random.int())` で合成できる（この呼び出しのみ `uses { Random }`）。

## Motivation

多くのプログラムが乱数を必要とするが、用途は 2 つに大別される。(1) 鍵・トークン・ソルトのような**予測不可能**な値（OS エントロピー、再現不可能であるべき）。(2) テスト・シミュレーション・手続き生成のような**再現可能**な擬似乱数（同じ種から同じ列）。

(1) は非決定的な外部作用なので effect（0009）として capability 追跡する。`wasi:random` はまさにこの暗号論的エントロピー源を WASI 0.2 の葉として供給する。(2) は決定論的なので effect にすると意味論に反する（再現性と非決定性は両立しない）。Emela は関数型であり、状態を値として引き回せるので、PRNG は `Prng` 値を受け取り「次の値と次の状態」を返す純関数として表現するのが自然である。これにより PRNG は capability を持たず、全 backend で同一結果を与える（0000）。

`Random` は 0009 の組み込み effect 集合の、`wasi:random` に対応する最小実体化である。

## Specification

以下、キーワードは RFC 2119 に従う。

### Part A: `Random` effect（`std.random`）

```
module random

effect Random {
    extern fn raw_int() -> Int
    extern fn raw_bytes(len: Int) -> Bytes

    pub fn int() -> Int { raw_int() }
    pub fn bytes(len: Int) -> Bytes { raw_bytes(len) }
}
```

- **P1（capability）**: `Random` を組み込み capability 集合（0009）に追加する。`Random` 操作は他の capability を要求しない（MUST）。`Random` は 0049 の意味でプリミティブ effect（`extern fn` バッキング）である。
- **P2（platform 関数）**: 各操作は 0013 の platform 関数に lower する。canonical 名は `random.raw_int` / `random.raw_bytes`。両者とも **infallible**（0043 の `throws` を持たない、MUST）。
- **P3（`int`）**: `Random.int()` は `Int` 全域にわたり一様に分布する値を返す（MUST）。`Int` は 0001 で 32bit 符号付きなので、負値を含む全 2^32 パターンが等確率である。値は暗号論的に安全な乱数源に由来する（下記 P5）。
- **P4（`bytes`）**: `Random.bytes(len)` は `len` バイトの `Bytes`（0051）を返す（MUST）。`len == 0` のときは長さ 0 の `Bytes` を返す。`len < 0` は実装定義（ホストが空を返してよい、MAY）。
- **P5（品質）**: `Random` の供給する乱数は暗号論的に安全（CSPRNG 品質）でなければならない（MUST）。`wasi:random/random` はこれを規定しており、`emela run` のホストも同等の OS エントロピー源を用いる（0052 parity）。予測可能・再現可能であってはならない（MUST NOT）。

### Part B: シード可能 PRNG（`std.random`、純粋）

```
pub record Prng { state: Int }
pub record Draw { value: Int, next: Prng }

pub fn seed(s: Int) -> Prng
pub fn next(g: Prng) -> Draw
```

- **Q1（純粋・決定論的）**: `Prng` / `Draw` は通常の record（0006）である。`seed` / `next` は capability を持たない純関数（`uses {}`、MUST）であり、同じ `Prng` からは常に同じ `Draw` を返す（決定論、MUST）。したがって全 backend で同一の列を生成する（0000、MUST）。
- **Q2（`seed`）**: `seed(s)` は種 `s` から `Prng` を構築する。実装は xorshift32 を用いるため、内部状態が 0 になる種は許されない。`s == 0` は固定の非ゼロ定数へ写像する（MUST）。それ以外の `s` はそのまま初期状態とする。
- **Q3（`next`）**: `next(g)` は生成器を 1 ステップ進め、`Draw { value, next }` を返す。`value` は生成された `Int`（`Int` 全域、負値を含む）、`next` は次状態の `Prng`。純粋なので `g` は変化せず、後続の呼び出しは返された `next` を用いる（MUST）。
- **Q4（アルゴリズム）**: 本仕様は xorshift32 を規範アルゴリズムとする。1 ステップは、状態 `x` に対し `x ^= x << 13; x ^= x >>> 17; x ^= x << 5`（`>>>` は 0053 の論理右シフト）を適用し、その結果を新状態かつ出力とする（MUST）。0053 のビット演算・シフトの意味（`Int` = i32、mod 2^32）に従う。
- **Q5（種の合成）**: エントロピー由来の種が要るときは `seed(Random.int())` と書く。この式は `Random` を使う行のみ `uses { Random }` を持ち、生成された `Prng` 以降の `next` は純粋である（Q1）。

## Examples

```
import std.random
import std.io

-- 決定論的：同じ種は同じ列を与える（テスト・再現に使う）
fn dice_rolls(g: Prng) -> Int uses {} {
    let d1 = random.next(g)
    let d2 = random.next(d1.next)
    (d1.value & 7) + (d2.value & 7)
}

-- エントロピー由来の値（予測不可能）
fn token() -> Bytes uses { Random } {
    Random.bytes(16)
}

-- エントロピーで種をまき、以降は純粋に回す
fn main() -> Unit uses { Io, Random } {
    let g = random.seed(Random.int())
    Io.print(random.next(g).value)
}
```

## Compilation Notes

この節は非規範的である。

- **component backend（`wasi:random`）**: `random.raw_int` は `wasi:random/random.get-random-u64` を呼び、返り値の `u64` を `Int`（i32）へ narrow する（`i32.wrap_i64`）。`random.raw_bytes` は `get-random-bytes(len)` を呼び、返る `list<u8>` を `Bytes` 表現へコピーする。生成 component は `wasi:random/random` のみを import する（`Random.bytes` 使用時。`Io` 併用時は `wasi:cli` も）。
- **`emela run`（wasip1）**: ホストが OS エントロピー（`getrandom` 等）で `random.raw_int` / `random.raw_bytes` を供給する。`socket.*` と同じ host-import 方式。0052 の parity 規則により、component backend と同一の観測挙動（P3/P4/P5、Q は backend 非依存）に準拠する。
- **PRNG（Part B）**: 純 Emela（0053 のビット演算）で実装されるため backend 固有のグルーは不要で、全 backend（`wasm-wasip2` / `emela run` / `js-node`）で同一結果を与える。`registry` への追加も不要。
- **registry**: `crates/emela-codegen/src/platform.rs` の `platform_interface()` に `random.*`（capability `"Random"`）を追加する。

## Open Questions

- **範囲付き乱数（`0..n`）・シャッフル・重み付き抽選**: PRNG の上に純関数として stdlib に追加しうるが、本仕様は最小の `seed` / `next` に絞る。
- **`wasi:random/insecure` / `insecure-seed`**: 高速だが非暗号の源。`Random` は暗号品質に絞り、insecure 源は将来仕様。
- **PRNG の品質**: xorshift32 は一般用途に十分だが crush 級ではない。より強い生成器（PCG 等、64bit 状態）は `Int` が 64bit 化（0001 Open Question）した後に検討する。
- **`Int` 幅**: `random.raw_int` と PRNG は現行の `Int`=i32 に追従する。64bit 化時は再検討する。

## 他仕様との関係

- **0009（Effect Semantics）**: 組み込み effect に `Random` を追加する。
- **0013（Platform Functions）**: registry に `random.*`（infallible）を追加する。
- **0043（Fallible Platform Functions）**: `random.*` は fallible では**ない**（`Io` / `Clock` と同じ infallible）。
- **0049（Effects as Compile-Time DI）**: `Random` はプリミティブ effect（leaf）。PRNG は effect ではない純粋計算。
- **0051（Bytes）**: `Random.bytes` の返り値型 `Bytes` を用いる。
- **0052（Backend Targets）**: `random.*` は `wasm-wasip2`（`wasi:random`）と `emela run`（ホスト）の双方で供給し、同一の観測挙動に準拠する（parity）。
- **0053（Bitwise Operators）**: シード可能 PRNG（xorshift32）が本仕様のビット演算・論理右シフトを用いる。
- **0025（Capability Manifest）**: capability `"Random"`・修飾名 `random.*` が manifest に現れる。
