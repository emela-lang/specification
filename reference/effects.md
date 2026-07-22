# Effects (`uses`)

> **リファレンス** — 現在の規範。ここに書かれた規則が今の言語の振る舞いである。
> ここに至った経緯・動機は末尾 [Provenance](#provenance) の RFC を参照。

関数が要求する **capability（能力）** を、関数型の `uses` 節に **effect row** として書く。
effect は静的な契約であり、型検査後に消去され、**実行時表現もランタイムコストも持たない**。

キーワード MUST / MUST NOT / SHOULD / MAY / MUST は RFC 2119 に従う。

## 用語

| 用語 | 定義 |
|---|---|
| **effect / capability** | 関数が要求する能力（`Io`・`Socket` 等）。名前は大文字始まりの識別子。 |
| **effect row** | effect の集合。`uses` 節が持つ値。順序・重複は正規化され、集合として扱う。 |
| **プリミティブ effect（leaf）** | `extern fn` 操作を1つ以上持つ effect。実体をホスト（platform 関数）が供給する。 |
| **派生 effect** | `extern fn` を持たない effect。操作はシグネチャ宣言のみで、実体を **handler** が供給する。 |
| **handler** | 派生 effect の各操作を、依存 effect の上で実装する単位。trait の impl に対応する。 |
| **source row** | ある関数本体が発生させる effect の集合（派生 effect を含みうる）。人間が読む intent。 |
| **leaf row** | source row を **discharge** した結果。プリミティブ effect と、未解決の派生 effect（residue）からなる。 |
| **discharge** | row 中の各派生 effect を、その handler の依存 row で置換する固定点変換。 |
| **未解決 effect** | in-scope な handler を持たない派生 effect。境界に残るとエラーになる。 |
| **権限表** | プログラムがホストへ要求する import 集合。leaf row のプリミティブ要素に等しい。 |

## 1. モデル

中心テーゼ：**関数の `uses` 節（の leaf row）が、そのまま WASM モジュールがホストへ要求する import 集合＝サンドボックスの権限表になる**。これが二重の保証を与える。

- **静的**: 宣言なき副作用は effect 検査が型エラーとして拒否する。
- **動的**: import されていないホスト関数は、WASM サンドボックスが実行時にも呼ばせない。

effect は capability の **追跡と、境界での供給** である。派生 effect の handler は [trait の impl](traits.md) と同じく **静的に解決・単相化・消去** される（コンパイル時 DI）。限定継続を伴う実行時 effect handler（Koka 型）は採らない。

`throws E`（error channel）と `uses F`（capability channel）は **独立した2本の channel** である。error は `uses` に寄与せず、`throw` や `?` は effect を発生させない。`throws` は本章の対象外（→ [Errors](errors.md)）。

## 2. 関数型の effect row

関数型の完全形：

```text
(A, B) -> T throws E uses F
```

```ebnf
FnType  ::= '(' [Type (',' Type)*] ')' '->' Type Throws? Uses?
Throws  ::= 'throws' Type
Uses    ::= 'uses' Row
Row     ::= '{' [Name (',' Name)*] ('..' RowVar)? '}'  |  RowVar
RowVar  ::= "'" ident
```

- **関数型の表記** で `uses` を省略した型は `uses {}`（純粋）である (MUST)。型は明示的な契約であり、推論されない。`uses {}` は省略してよい。
- `throws E` と `uses F` はいずれも型の一部・public API の一部である。`uses` の異なる関数型は **異なる型** である。

```text
Path -> String                              -- pure, non-throwing
Path -> String uses { Fs }                  -- effectful, non-throwing
Path -> String throws IoError               -- throwing, pure
Path -> String throws IoError uses { Fs }   -- throwing, effectful
```

（**関数定義** の `uses` 省略は推論を意味する。§5 を見よ。省略の意味は「型の表記」と「定義」で異なる。）

## 3. effect の宣言

```emela
module io

effect Io {
    extern fn write_stdout(s: String) -> Unit
    extern fn write_stderr(s: String) -> Unit

    pub fn print<T: Show>(s: T) -> Unit { write_stdout(s.to_string()) }
    pub fn eprint<T: Show>(s: T) -> Unit { write_stderr(s.to_string()) }
}
```

- `effect Name { items }` はモジュール内のトップレベルアイテム（enum と同格）である。`Name` は大文字で始まらなければならない (MUST)。1 モジュールに複数の effect を宣言してよい (MAY)。
- `items` は `fn` / `pub fn` / `extern fn` を含んでよい。各操作は暗黙に `uses { Name }` を持ち (MUST)、操作に明示的な `uses` 節を書くことはエラー (MUST NOT)、`intrinsic fn` はエラー (MUST NOT)。
- `extern fn` を1つ以上持つ effect は **プリミティブ effect** である。その `extern fn` の実体はホストの platform 関数が供給する（→ [Runtime Boundary](runtime-boundary.md)）。canonical platform 名は **モジュール名** で修飾する（例：`io.write_stdout`）。effect 名 `Io` は row・呼び出し構文・診断にのみ現れ、backend シンボルや capability 検証には影響しない (MUST)。
- `extern fn` を持たない effect は **派生 effect** である（§8）。
- 1つの effect はプリミティブか派生のいずれか一方である (MUST NOT: 混在)。
- extern バッキング操作は同じ effect ブロック内からのみベア名で呼べる。ブロック外からは参照できない (MUST NOT)。

## 4. effect 操作の呼び出しと `uses` ゲート

effect 操作は **修飾形 `Name.op(...)` でのみ** 呼べる (MUST)。ローカル宣言か import かを問わず一様であり、ベア名 `op(...)` で effect 操作に解決してはならない (MUST NOT)。

`Name.op(...)` が解決されるのは、次の **両方** が成り立つときである (MUST)。

1. `Name` がスコープ内の effect（同一ファイル宣言 ∪ import 済み公開 effect）に解決できる。
2. 語彙的に囲む最も内側の `uses` 保持スコープ（関数定義・関数リテラル）の宣言 `uses` row に `Name` が含まれる。

条件を満たさない場合、原因ごとに別の診断を出す (MUST)。

```emela
fn f() -> Unit uses { Io } {
    Io.print("hi")       -- OK
}

fn g() -> Unit uses { Io } {
    print("hi")          -- error: `print` は effect `Io` の操作。`Io.print(...)` と呼ぶ
    Io.write_stdout("x") -- error: `write_stdout` は `Io` の非公開操作
}

fn h() -> Unit {
    Io.print("hi")       -- error: effect `Io` は `h` の `uses` に宣言されていない
}
```

`Io` が in-scope でない（import が無い）場合は `unknown effect` エラーとし、その effect を宣言するモジュールが既知なら import を案内する (SHOULD)。

`uses { Name }` の各名前も、スコープ内の effect に解決されなければならない (MUST)。未知の名前はエラーである。

### 名前解決順位

パス `p.x` の先頭 `p` は次の順で解決する。

1. ローカル束縛 → フィールドアクセス／レシーバ呼び出し
2. 大文字始まりならスコープ内の effect → effect 操作呼び出し
3. import 済みモジュールの修飾子 → 修飾関数呼び出し

enum variant・型変換は `::` であり `.` とは衝突しない。import は **モジュール単位** で、effect 操作の関数単位 import は存在しない（→ [Modules & Imports](modules.md)）。

## 5. 推論と注釈

- **関数定義**（名前付き `fn`・無名関数）で `uses` を省略した場合、effect row は本体から **推論** される (MUST)。推論される row は、呼び出し・guard・分岐の和集合に従って計算される **最小の row** である。
- 再帰・相互再帰は最小不動点で解く。各関数の row を `{}` から始め、変化がなくなるまで和集合を反復する。effect の全体集合は有限なので停止する。
- `pub fn` は `uses` を **明示しなければならない** (MUST)。public API の effect は推論結果ではなく宣言である。`extern fn` / `intrinsic fn` も常に明示である。
- `uses` 注釈は **上界** である。推論 row を `I`、宣言 row を `D` として `I ⊆ D` でなければならない (MUST)。`I ⊄ D` はエラーで、不足 effect を列挙すべきである (SHOULD)。過大宣言 `D ⊋ I` は合法である（将来 effect を追加する予約に使える）。過大宣言は権限表を増やさない（§9）。

```emela
fn load() -> String {                    -- uses { Fs } と推論（private は注釈不要）
    Fs.read("config.txt")
}

pub fn main() -> Unit uses { Io, Fs } {  -- pub は明示必須
    Io.print(load())                     -- 推論 { Io, Fs } ⊆ 宣言 { Io, Fs }
}

fn bad() -> Unit uses { Io } {
    Fs.read(path)                        -- error: 推論 { Fs } ⊄ 宣言 { Io }
}
```

## 6. Subsumption と row 拡張

### Subsumption（部分集合による適合）

具体 row `F1 ⊆ F2` のとき、型 `(A) -> T uses F1` の関数値は、型 `(A) -> T uses F2` が期待されるあらゆる位置（引数渡し・注釈付き束縛・戻り値・record フィールド）に適合する (MUST)。

```text
F1 ⊆ F2
─────────────────────────────────────────
(A) -> T uses F1  <:  (A) -> T uses F2
```

関数型がネストする場合は通常の変性に従う。**パラメータ位置の row は反変**（より大きい row を受ける関数は、より小さい row を期待する位置に適合）、**結果位置は共変**。

```emela
fn twice(f: (Int) -> Int uses { Fs, Net }) -> Int uses { Fs, Net } { ... }
fn pure_inc(x: Int) -> Int { x + 1 }     -- uses {}
twice(pure_inc)                          -- OK: {} ⊆ { Fs, Net }
```

subsumption は同一性ではなく **適合**（部分型方向の変換）である。「`uses` の異なる関数型は異なる型」は維持される。適合は実行時表現を持たず、coercion コードは生成されない。

### Row 拡張

```text
uses { Io, ..'e }     -- 意味: { Io } ∪ 'e
```

具体 effect と row 変数を並べた拡張 row を書ける。単一化は **最小解** を取る。具体 row `C` を `{ Io, ..'e }` に対応付けるとき `'e = C \ { Io }` とする (MUST)。

```emela
fn traced<T, U>(x: T, f: (T) -> U uses 'e) -> U uses { Log, ..'e } {
    Log.info("call")     -- uses { Log }
    f(x)                 -- uses 'e
}
```

## 7. effect-row 多相

`map` / `filter` / `fold` のような高階関数を、純粋・effectful の両方に同一定義で適用するため、`uses` を **effect-row 変数** で受け取れる。

```emela
fn map<T, U>(xs: Array<T>, f: (T) -> U uses 'e) -> Array<U> uses 'e { ... }

map([1, 2, 3], fn (x: Int) -> Int { x + 1 })      -- 'e = {} → map 全体 uses {}
map(paths, fn (p: Path) -> String uses { Fs } { Fs.read(p) })  -- 'e = { Fs }
```

- effect-row 変数は sigil `'` を付けた識別子（`'e`）で書く。**型パラメータ `<...>` とは別カテゴリ** であり、`<...>` に書いてはならない (MUST NOT)。
- `'e` は **`uses` の位置にのみ** 現れる (MUST)。引数型・戻り値型・`throws` 型には書けない。
- `'e` は宣言不要で、`uses` 位置に現れた時点でシグネチャに暗黙に全称量化される。スコープはその関数定義に閉じ、異なる関数の同名 `'e` は無関係である。
- 各 effect-row 変数は、少なくとも1つのパラメータの `uses` 位置に現れなければならない (MUST)。どの `uses` 位置にも現れない row 変数は推論できずエラーである。
- 呼び出し時、`'e` は実引数の関数値の effect row から推論する。呼び出し式全体の effect row は、代入を関数自身の `uses` 節へ適用したものになる。
- effect row は静的契約で実行時表現を持たないため、effect-row 多相は **単相化を必要としない**。型検査後に消去するだけでよく、`'e` の違いだけで別の特殊化を生まない。

subsumption（§6）は v1 では **具体 row 同士の適合にのみ** 適用し、row 変数を通した緩和（bounded row variable）はしない (MUST)。

## 8. handler・discharge・erasure

**派生 effect** の操作は本体を持たないシグネチャ宣言であり、実体を **handler** が供給しなければならない (MUST)。

```emela
effect Log {                 -- 派生 effect（extern を持たない）
    pub fn info(msg: String) -> Unit
    pub fn error(msg: String) -> Unit
}

handler for Log uses { Io } {          -- 依存 effect Io の上で実装
    fn info(msg: String) -> Unit  { Io.write_stdout("[INFO] " ++ msg ++ "\n") }
    fn error(msg: String) -> Unit { Io.write_stderr("[ERROR] " ++ msg ++ "\n") }
}
```

- `handler for E uses { R } { items }` は派生 effect `E` の **default handler** を宣言する。`E` のすべての操作にちょうど1つ、シグネチャが厳密に一致する実装を与えなければならない (MUST)。
- handler 本体が使用する effect は依存 row `R` に含まれていなければならない (MUST)。`R` は派生 effect を含んでよく (MAY)、その場合 discharge は推移的に進む。
- 派生 effect `E` の **default handler は `E` を宣言したモジュールに置かれた handler** である。`E` が import 済みなら、その default handler も自動的に in-scope とみなす (SHOULD)——利用者は effect を import すれば handler のための追加 import を要しない。
- handler は第一級の値ではない (MUST NOT)。束縛・受け渡し・戻りはできない。解決の対象であって実行時表現を持たない。

### 未解決 effect

in-scope な default handler を持たない派生 effect を **未解決 effect** と呼ぶ。未解決 effect の操作呼び出し **自体はエラーにしない** (MUST NOT)。その effect は discharge されずに row に蓄積し、呼び出し元へ伝播する。エラーになるのは境界だけである（§9 D6）。これにより handler より先に effect とその利用側を書ける。

### 解決・discharge・erasure

- **D1（使用箇所）**: 派生 `E` の `E.op(...)` は、default handler の `op` 実装をコンパイル時に一意に選び、その本体への **直接 call へ単相化置換** する（trait メソッド discharge と同じ機構）。handler 本体がさらに派生 effect を呼べば再帰的に解決する。到達可能な特殊化は有限でなければならない。
- **D2（row の discharge）**: 派生 `E` の操作呼び出しは呼び出し元の **source row** に `E` を寄与する（§4 の `uses` ゲートは `E` に対して従来どおり働く）。コンパイラは、handler を持つ各派生 effect を、その handler の依存 row `R` で置換する discharge を **固定点** まで繰り返す。結果が **leaf row** である。

  ```text
  source row  S  = 本体が発生させる effect の union（派生を含む）
  discharge      = handler を持つ派生 E を deps(handler(E)) で置換、固定点まで反復
  leaf row  L    = discharge(S) = { プリミティブ } ∪ { 未解決の派生（residue） }
  ```

- **D3（erasure）**: 派生 effect・handler・その解決は typed IR 化の前に **完全に消去される** (MUST)。IR に残るのはプリミティブ effect の platform 呼び出しだけである。辞書渡し・evidence・実行時ディスパッチは生成しない。
- **D6（境界の全解決）**: プログラムのエントリ（`main`）および executable 境界では、その関数の **leaf row はプリミティブ effect のみからなっていなければならない** (MUST)。residue（未解決の派生 effect）が残ればコンパイルエラーであり、未解決 effect 名と handler が期待されるモジュールを示す (SHOULD)。中間の関数は未解決 effect を row に持ってよい (MAY)——エラーは境界でのみ生じる。

```emela
import std.io
import std.log

fn greet(name: String) -> Unit uses { Log } {
    Log.info("hello, " ++ name)
}

fn main() -> Unit uses { Log } {
    greet("emela")
    -- source row : { Log }
    -- discharge  : Log -> { Io }（handler の依存）
    -- leaf row   : { Io } ← 権限表・manifest はこれ
}
```

`std.log` に handler が無ければ、中間コードは通るが `main` で露見してエラーになる（leaf row に `Log` が residue として残る）。

## 9. 権限表：source row と leaf row

- **権限表を成すのは discharge 後の leaf row のプリミティブ要素** である。source row（派生を含みうる）は人間が読む intent の宣言であり、実際にホストへ要求する import 集合はプリミティブな leaf である。
- capability manifest は到達可能な platform 呼び出しから計算される（`uses` 非依存）。discharge 後に残るのは leaf の platform 呼び出しだけなので、manifest は自動的に leaf のみを列挙する。**派生 effect は manifest に現れない**（純 Emela の抽象であり、新たなホスト権限を生まない）。
- したがって過大宣言（§5）は Runtime 要求を増やさない。要求集合は宣言 row ではなく到達 platform 関数から計算される。

（manifest とカバレッジ検査「要求 ⊆ 供給」の詳細は → [Runtime Boundary](runtime-boundary.md)。）

## 10. 組み込みプリミティブ effect

ホストが供給するプリミティブ effect。名前は大文字始まり。操作・platform 名・供給契約は各 effect のモジュールと [Runtime Boundary](runtime-boundary.md) が定める。

| effect | 主な操作 | backing | 備考 |
|---|---|---|---|
| `Io` | `write_stdout` / `write_stderr`（+ `print` 等 default 操作） | platform | 標準出力・標準エラー |
| `Clock` | 時刻取得 | platform | — |
| `Socket` | `listen` / `accept` / `read` / `write` / `close` | platform（`wasi:sockets`） | 平文 TCP のみ。TLS は含まない。`Bytes` を境界型とする。予約語 `net` の TCP 部分を実体化 |
| `Http` | HTTP クライアント | platform | TLS がホスト責務のため **プリミティブのまま**（`Socket` の上に載せない） |

派生 effect の例として `HttpServer`（`Socket` の上の handler。accept ループはゲスト、HTTP パースは stdlib）がある。`Log` も派生 effect として `Io` の上に実装できる。

`host.*` は embedder（ホスト実装者）が定義する capability の名前空間である（→ [Runtime Boundary](runtime-boundary.md) の `host.*`）。

## 未解決事項

現行のリファレンスで **まだ確定していない** 点。実装・利用時はこれらに依存しない。

- **組み込みプリミティブ集合の確定形**。`Fs`・`Random`・`Net` は初期の予約（旧 `fs`/`random`/`net`）に由来するが、大文字表記・realization・プリミティブ/派生の別を固定する現行仕様がまだ無い（`Net` の TCP 部分は `Socket` として実体化済み）。
- **スコープ限定の handler 差し替え（DI の本体）と coherence**。default（同一モジュール）と異なる handler の注入（テスト用モック、呼び出しグラフ全体への handler スレッディング）、同一 effect に複数 handler が現れる場合の大域一意性。
- **状態を持つ handler**（コネクションプール等）を erasure と両立させる方法。
- **mixed effect**（同一 effect が一部 extern・一部 handler）の是非（当面禁止）。
- **bounded row variable**（`'e ⊆ { Fs, Net }`）と、row 変数を通した subsumption。
- **error-row 多相**（`throws { A, ..'r }`）の導入可否（→ [Errors](errors.md)）。
- **`throws` の推論**（private 関数で本体から推論する）。
- **無名関数が effect-row 多相を持てるか**。

## Provenance

この章は次の RFC を現在形に畳み込んだものである。経緯・却下案・詳細な動機はそちらを見よ。

- 中心テーゼ・原則: [`../specs/0000`](../specs/0000-language-design-principles.md)
- 関数型と2 channel: [`../specs/0008`](../specs/0008-function-types-with-effects.md)
- effect 意味論・伝播・組み込み集合: [`../specs/0009`](../specs/0009-effect-semantics.md)
- effect-row 多相: [`../specs/0022`](../specs/0022-effect-row-polymorphism.md)
- 推論・subsumption・row 拡張: [`../specs/0023`](../specs/0023-effect-inference-and-subsumption.md)
- effect 宣言（初出）: [`../specs/0036`](../specs/0036-effect-declarations-and-effect-operations.md)（[`0037`](../specs/0037-module-imports-and-first-class-effects.md) が置換）
- first-class effect・モジュール単位 import・`uses` ゲート: [`../specs/0037`](../specs/0037-module-imports-and-first-class-effects.md)
- handler・discharge・erasure（コンパイル時 DI）: [`../specs/0049`](../specs/0049-effects-as-compile-time-di.md)
- `Socket` プリミティブ effect: [`../specs/0050`](../specs/0050-socket-primitive-effect.md)
