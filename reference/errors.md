# Errors (`throws`)

> **リファレンス** — 現在の規範。経緯・動機は末尾 [Provenance](#provenance) の RFC を参照。

回復可能な失敗は、関数型の `throws E` 節と `throw` 式で表す。`throws` は
[`uses`（capability channel）](effects.md)とは **独立した error channel** である。
error は `uses` に寄与せず、`throw` や `?` は effect を発生させない。

キーワード MUST / MUST NOT / SHOULD / MAY は RFC 2119 に従う。

## 用語

| 用語 | 定義 |
|---|---|
| **`throws E`** | 関数が送出しうる error の型 `E` を宣言する節。省略時は `throws Never`（非送出）。 |
| **`throw e`** | error `e` を送出する式。型は `Never` で、任意の期待型に適合する。 |
| **throwing な呼び出し** | `throws E`（`E ≠ Never`）を持つ関数の呼び出し。 |
| **`?`** | throwing な呼び出しの error を現在の関数の `throws` へ伝播する後置 operator。 |
| **`try` / `catch`** | 送出された error を捕捉してハンドルする式。 |
| **`panic`** | 回復不能な異常終了。どの channel にも現れない。 |
| **`Never`** | 値を持たない型。`throws Never` は error を送出しえない。 |

## 1. 失敗の3分類

| 種類 | 表現 | 用途 |
|---|---|---|
| 回復可能な失敗 | `throws E`（error channel） | 呼び出し元が対処しうる失敗 |
| 値の不在 | `Option<T>`（通常の値） | 「無い」を返す。→ Option は Core Prelude の enum（[`specs/0042`](../specs/0042-option-as-core-prelude-enum.md)） |
| 回復不能な異常 | `panic` | プログラムを異常終了させる |

回復可能な失敗を `Result<T, E>` で表さない。`Result` は組み込み型ではない。

## 2. `throws` と `throw`

関数が送出しうる error の型は、戻り値型に続く `throws E` 節で宣言する。

```emela
fn read(path: Path) -> String throws IoError uses { Fs }
fn parse(input: String) -> Ast throws ParseError
```

- `throws E` を持たない関数は error を送出しない。これは `throws Never` と等価である。
- `E` は **単一の型** である (MUST)。複数種類の error は enum で束ねる。
- capability を `throws` に書くことはできない (MUST NOT)。`throws` と `uses` は別 channel である。
- `throw e` は通常の値を返さない。型は `Never` であり、任意の期待型に適合する。
- `throw e` が現れる関数は `throws E` を宣言していなければならず (MUST)、`e` の型は `E` に適合していなければならない。ただし `throw e` が `try` block 内にあるときは `catch` に捕捉される（§5）。
- `throw` は capability ではないため `uses` には現れない。

## 3. throwing な呼び出しの扱い

throwing な呼び出しは、次のいずれかの位置に置かなければならない (MUST)。満たさない throwing な呼び出しはコンパイルエラーである。

1. `?` を付けて error を呼び出し元へ伝播する。
2. `try` block 内に置き、`catch` で捕捉する。

## 4. `?` operator

`?` は throwing な呼び出しの error を、現在の関数の `throws` 節へ伝播する短絡評価 operator である。`throw`・早期 return に対する糖衣と捉えてよい。

```emela
let value = expr?
-- ≡
let value = try { expr } catch { e -> throw e }
```

- `expr?` を使う関数の戻り値型は `throws E` を宣言していなければならない (MUST)。
- `?` が伝播する error 型は、現在の関数が宣言する error 型と **一致** していなければならない (MUST)。異なる場合は `try`/`catch` で明示的に変換する（`From`/`Into` 的な error 変換は未導入 → [未解決事項](#未解決事項)）。
- `?` は throwing な呼び出し専用であり、`Option<T>` には使えない (MUST NOT)。`Option` は `match` またはコンビネータで扱う。error channel と `Option` の間に暗黙変換は無い。不在を error として伝播するには `Option.ok_or(o, e) -> T throws E` で明示的に橋渡ししてから `?` を使う。
- `?` は対象の式を1回だけ評価し、成功なら値を取り出し、error なら現在の関数の `throws` へ短絡する。後続の式は評価されない。
- `?` 自体は effect を発生させない。`expr?` の effect は `expr` の effect と同じである。

```emela
fn load(path: Path) -> Config throws IoError uses { Fs } {
    let text = read(path)?    -- read: throws IoError → そのまま伝播
    parse(text)?
}
```

## 5. `try` / `catch` 式

`try { block } catch { arms }` は式である。

- `block` を評価する。成功値 `v` で完了すれば式全体の値は `v` であり、`catch` は評価されない。
- `block` の評価中に error `e` が送出されたら、`e` を `catch` の arm に上から順に pattern match する。一致した arm の本体を評価し、その値が式全体の値になる。
- `try` block 内では throwing な呼び出しに `?` を付ける必要はない。送出された error はすべて `catch` に routing される。
- `catch` の arm は、`block` 内で送出されうる error 型を **網羅** していなければならない（`match` の exhaustiveness と同じ、MUST）。wildcard arm を用いてよい。
- 式の型は、`block` の成功値の型とすべての `catch` arm 本体の型の共通型である。全分岐は同じ型を返さなければならない。
- **error channel の解消**: `block` 内の error を `catch` がすべてハンドルすれば、`try`/`catch` 式自体は error を送出しない。したがって `throws` を宣言しない関数でも、throwing な呼び出しを `try`/`catch` で包めば呼べる。arm 本体で再度 `throw` すれば再送出でき、その場合は関数が対応する `throws` を宣言していなければならない。
- effect は `block` と到達しうる `catch` arm 本体の effect の和集合である。

```emela
fn loadOrDefault(path: Path) -> Config uses { Fs } {
    try {
        parse(read(path))
    } catch {
        IoError::NotFound -> defaultConfig()
        e -> panic("unrecoverable")
    }
}
```

## 6. panic

`panic` は組み込みの回復不能エラーである。現在の評価を終了し、プログラムを回復不能な異常終了へ移行させる。

- `panic` は通常の値を返さず、型は `Never` で任意の期待型に適合する。
- 回復可能な失敗に `panic` を使ってはならない (MUST NOT)。回復可能な失敗は `throws` または `Option` で表す。
- `panic` は `throws` でも `uses` でもなく、どちらの channel にも現れない。
- WASM / WAMR では panic を runtime trap または runtime-provided panic handler に lower してよい。embedder は panic message をログに出して異常終了してよいが、通常評価へ復帰してはならない (MUST NOT)。

## 7. エントリポイント

プログラムのエントリ `main` の `throws` は `Never` でなければならない (MUST)。`main` は `throws` 節を省略するか `throws Never` を宣言する。`Never` 以外を宣言するとコンパイルエラーである。`main` 本体が throwing な呼び出しや `throw` を含む場合、`main` 自身が `try`/`catch` で処理するか `panic` で異常終了させなければならない。error を `main` の外へ伝播することはできない。

## 8. ホストの失敗（fallible platform 関数）

ホスト（platform 関数）が報告する失敗も、この `throws` channel に流れる（→ [Runtime Boundary](runtime-boundary.md)）。

- registry のエントリと `extern fn` は `throws E` を宣言してよい (MAY)。宣言しないエントリは失敗しない（`throws Never` と等価）。
- `E` は、その extern を宣言する core モジュール内で宣言された、型パラメータを持たない `pub enum` でなければならない (MUST)。
- 呼び出し側から見て通常の Emela error であり、§3–§5 の規則がそのまま適用される。新しい構文・型は導入しない。
- `throws` の有無は `uses` row・カバレッジ検査・[capability manifest](runtime-boundary.md) に一切影響しない (MUST)。error は capability ではない。
- ホストは1回の呼び出しにつき、成功値または error 値の **ちょうど一方** を返さなければならない (MUST)。想定内の失敗で trap / panic してはならない (MUST NOT)。trap は ABI 契約違反に限る。

```emela
effect Fs {
    extern fn read(path: String) -> String throws FsError
}
```

## 未解決事項

- **union error 型**（`throws A | B`）を許すか。
- **error 型の暗黙変換**（`From` / `Into`、`?` 演算子の trait 化）を導入するか。
- `try` block 内から `catch` を飛び越えて関数の `throws` へ伝播する手段（`?` の許可）。
- `throws` の **推論**（private 関数で `throws` を省略し本体から推論する）。→ [Effects の推論](effects.md#5-推論と注釈) の error 版。
- 「不在」を返す platform 関数（環境変数取得等）に `Option<T>` 戻りの規約を並置するか。

## Provenance

- error handling（`throws` / `throw` / `?` / `try`–`catch` / `panic` / `main`）: [`../specs/0011`](../specs/0011-error-handling.md)
- fallible platform 関数（ホスト失敗の `throws` 接続）: [`../specs/0043`](../specs/0043-fallible-platform-functions.md)
- `throws` と `uses` の直交: [`../specs/0009`](../specs/0009-effect-semantics.md)、[`../specs/0008`](../specs/0008-function-types-with-effects.md)
