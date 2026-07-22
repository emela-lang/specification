# Runtime Boundary

> **リファレンス** — 現在の規範。経緯・動機は末尾 [Provenance](#provenance) の RFC を参照。

コンパイラは副作用も純粋プリミティブ演算の意味も **実装しない**。実体は境界の外側が供給する。
コンパイラが保持するプリミティブ知識は **(a) 値の表現**（`Int` = i32 等）と
**(b) intrinsic → 命令の対応表** の2つだけである。

境界は2種類ある。

| 境界 | 宣言 | 供給者 | lower 先 | 純度 |
|---|---|---|---|---|
| **platform 関数** | `extern fn` | Runtime（ホスト） | Runtime への **呼び出し** | effectful（[capability](effects.md)を発生） |
| **intrinsic** | `intrinsic fn` | backend | native 命令へ **インライン** | 純粋（常に `uses {}`） |

キーワード MUST / MUST NOT / SHOULD / MAY は RFC 2119 に従う。

## 用語

| 用語 | 定義 |
|---|---|
| **platform 関数** | 副作用の実体を Runtime が供給する境界関数。[capability effect](effects.md) の唯一の源。 |
| **`extern fn`** | platform 関数の宣言。本体を持たない。 |
| **`intrinsic fn`** | 純粋プリミティブ演算の宣言。本体を持たず、backend が命令へ lower する。 |
| **registry** | 言語仕様が定める platform 関数の標準インターフェース集合。 |
| **カバレッジ検査** | 「プログラムの要求 ⊆ backend の供給」を静的に検査する。 |
| **capability manifest** | 生成物に埋め込む、要求権限の機械可読な宣言。 |
| **Core Prelude** | 全コンパイル単位に暗黙 import されるコアモジュール。 |
| **`host.*`** | embedder（ホスト実装者）が定義する capability の名前空間。 |

## 1. platform 関数（副作用の唯一の源）

[capability effect](effects.md) を発生させる手段は platform 関数の呼び出しのみである。コンパイラは capability effect を持つ組み込み演算を提供してはならない (MUST NOT)。

- **registry**: 本仕様が platform 関数を列挙する。各エントリは修飾名（`io.write_stdout`）・引数型・戻り値型・capability を持つ。個々のプログラムが任意に定義するものではない。
- **`extern fn` 宣言**: Emela コードは platform 関数を `extern fn` で宣言し、本体を持たない (MUST NOT)。シグネチャは registry エントリと一致しなければならない (MUST)。registry に無い platform 関数の `extern fn` はコンパイルエラーである (MUST)。backend を名指し / 分岐する構文は存在しない (MUST NOT)——ゆえに `extern fn` を含むソースは backend 非依存である。
- **供給契約**: 各 target / backend は registry の **部分集合** を実装してよい (MAY)。executable が、選択された target の提供しない platform 関数を推移的に要求する場合、コンパイルはエラーにしなければならない (MUST)。エラーは不足関数を明示すべきである (SHOULD)。
- **canonical 名**: platform 名は **モジュール名** で修飾する（`io.write_stdout`）。effect 名（`Io`）は [`uses` row・呼び出し構文・診断](effects.md#3-effect-の宣言)にのみ現れ、backend シンボル・capability 検証には影響しない。

慣習として `extern fn` は標準ライブラリ（[effect ブロック](effects.md#3-effect-の宣言)内）が宣言し、`pub fn` で包む。アプリケーションはラッパーを通じて使い、`extern fn` を直接書く必要はない。

## 2. 失敗する platform 関数

registry エントリと `extern fn` は `throws E` を宣言してよい (MAY)。ホストの失敗は通常の [Emela error](errors.md#8-ホストの失敗fallible-platform-関数) として呼び出し側に届く。`throws` の有無は `uses` row・カバレッジ検査・manifest に影響しない (MUST)。詳細は [Errors §8](errors.md#8-ホストの失敗fallible-platform-関数)。

## 3. カバレッジ検査

- capability effect の発生源は platform 関数のみであるから、プログラムが実行しうる effect 全体は、`main` から推移的に到達可能な platform 関数の集合に等しい。これが Runtime が解決すべき契約である。
- 各 backend は供給する platform 関数（および intrinsic）の集合を宣言し、コンパイラは **要求 ⊆ 供給** を検査する。不足があればエラーにする (MUST)。
- library（`main` を持たない compilation unit）は、要求集合を metadata として保持してよい (MAY)。

（派生 effect を含む row の discharge 後にこの検査が働く点は [Effects §8–§9](effects.md#8-handlerdischargeerasure) を参照。）

## 4. intrinsic（純粋演算）

純粋なプリミティブ演算は `intrinsic fn` で宣言する。構文は `extern fn` と同型で本体を持たない。

```emela
intrinsic fn i32_add(a: Int, b: Int) -> Int uses {}
intrinsic fn array_get_unchecked<T>(a: Array<T>, i: Int) -> T uses {}
```

- `intrinsic fn` は **純粋** でなければならず、常に `uses {}` である (MUST)。capability effect を持ってはならない。本体を書けない (MUST NOT)。backend 名指し構文は無い (MUST NOT)。
- 呼び出しは backend が **native 命令または式へインライン** に lower する（platform 関数が Runtime への呼び出しに lower するのと対照的）。同一の intrinsic 名は、どの backend でも観測可能な意味が一致するように lower されなければならない (MUST)。
- ジェネリック intrinsic は引数型から型引数を推論して単相化される。型変数は typed IR に到達する前に具体型へ置換される。
- intrinsic の集合もカバレッジ検査（§3）の対象である。
- 演算子（[Traits の演算子 trait](traits.md#6-演算子の-trait-化)）の組み込み型実装、`Char`/`String` 変換（`char_from_code` 等）、`Array` 基本操作（`array_length<T>` 等）はすべて intrinsic として供給される。

## 5. Core Prelude

コンパイラは、指定するコアモジュール（`std.core`）を全コンパイル単位に **暗黙に import** する (MUST)。Core Prelude は特別なコンパイラ組み込みではなく通常の Emela ソースである。

- Core Prelude は[演算子 trait](traits.md#6-演算子の-trait-化)（`Add`/`Sub`/…/`Eq`/`Ord`/`Concat`）と `Monoid`、および `Int`/`Float`/`String`/`Char` に対するそれらの `impl`（本体が `intrinsic fn` を呼ぶ）を置く。この暗黙 import により `1 + 2` やリテラルの `Int` 型が明示 import なしに解決する（後方互換）。
- Core Prelude が宣言する `intrinsic fn` は、**全コンパイル単位からベア名で可視** である (MUST)。prelude はどこにも暗黙 import されるため、そこで宣言された純粋 intrinsic はモジュール境界で遮られない（platform `extern` の module-private 規律とは異なる）。`char_from_code` / `string_from_char` / `array_length` / `array_get` / `array_push` が無 import のベア名で使える。
- prelude 以外のモジュールが宣言する intrinsic は、従来どおりその宣言モジュール内でのみベア可視であり、公開 API は `pub fn` ラッパが担う。
- 名前解決は[通常の import](modules.md#3-名前解決順位)に従い、ローカル束縛・同一 entry の関数は prelude を shadow できる。

```emela
fn main() -> Int uses {} {
    1 + 2
    -- 1 + 2 →(Traits) Add.add(1, 2) → impl Add for Int → i32_add(1, 2) →(backend) i32.add
}
```

## 6. capability manifest

コンパイラは executable の **要求集合** を静的に確定し、生成物に機械可読な manifest として埋め込む。ホストはインスタンス化の前に要求権限を監査できる。「[effect row = 権限表](effects.md#9-権限表source-row-と-leaf-row)」をツールチェーン全体の契約にする。

- **要求集合の計算**: `main` から推移的に到達可能な platform 関数（`requires`）、その capability（`capabilities`）、intrinsic（`intrinsics`）。**実際の到達可能性** から計算し、`uses` の過大宣言は要求集合を増やさない。
- **埋め込み**: WASM 出力では custom section `emela:capabilities` に JSON として格納する (MUST for executables)。非 WASM backend は同内容を添付すべきである (SHOULD)。
- **真実性（MUST）**: manifest は計算された要求集合と正確に一致しなければならない。過少申告・過大申告のいずれも禁止。
- **決定性（MUST）**: 配列はソート済み・重複なし、キー順固定、余分な空白なし。同一プログラムから同一バイト列を生成する（reproducible build）。

```json
{"format":1,"requires":["io.write_stdout"],"capabilities":["Io"],"intrinsics":["string_concat"],"entry":true}
```

- **ホスト側の利用**: ホストは manifest を読み、`requires ⊆ 供給` でないモジュールのインスタンス化を拒否できる。これはコンパイル時カバレッジ検査（§3）の配布後の対である。manifest 監査は WASM の import 解決を置き換えない——最終的な強制は常にサンドボックス側にある（[二重保証](effects.md#1-モデル)）。

## 7. embedder 定義 capability（`host.*`）

embedder は標準 registry を `host.*` 名前空間で拡張できる。デバイス固有の API（GPIO・センサー・独自プロトコル）を、標準 capability と同じ規律の下で公開する。

- capability 識別子 `host.<ident>` を導入する。`host` は単独では使用できず (MUST NOT)、常に `host.<cap>` の形で用いる（`uses { host.gpio }`）。標準 registry のモジュールパスと `host.*` は互いに素である (MUST)。
- embedder は **host interface package**（`host.` 配下のモジュールに `extern fn` を宣言した Emela ソース群）をビルド入力として供給できる (MAY)。`extern fn` の規則は §1 と同一で、宣言する capability はモジュールパスに対応する `host.<ident>` でなければならない (MUST)。
- `host.*` の platform 関数はカバレッジ検査（§3）と [manifest](#6-capability-manifest)（§6）の対象である (MUST)。package を供給しないビルドで `host.*` を参照すれば未定義エラーである。
- `host.*` を参照しないソースの backend 非依存性は変化しない。`host.*` を参照するソースは、その host interface を供給する embedder に対してのみ有効な **宣言された特殊化** であり、manifest に明示される。

## 未解決事項

- registry 初期セットの確定（`io` と `log` の分離、capability の割り当て）。
- **値マーシャリング ABI** をどこまで本仕様に固定し、どこから backend ABI に閉じるか（error payload のアロケータ契約を含む）。
- `extern fn` / `intrinsic fn` をアプリケーションでも宣言可能にするか、std に限定するか。
- Core Prelude の供給方法（ソース埋め込み vs `--package` 供給）と、`Show` / `to_string` を prelude に含めて無 import 化するか。
- capability の粒度（`fs` の read/write 区分、`host.gpio` の分割）を manifest format 2 で入れるか。
- host interface package のバージョニングと、広く使われる host API の標準 registry への昇格プロセス。
- backend target ごとの供給集合と import 形（`wasm-unknown` / `wasm-wasip2` 等）の詳細は [`specs/0052`](../specs/0052-backend-targets.md)。

## Provenance

- platform 関数・registry・extern・供給契約・カバレッジ検査: [`../specs/0013`](../specs/0013-platform-functions-and-runtime-provision.md)
- intrinsic・Core Prelude・演算子/変換/配列演算の移設: [`../specs/0021`](../specs/0021-intrinsics.md)
- fallible platform 関数（`throws`）: [`../specs/0043`](../specs/0043-fallible-platform-functions.md)（→ [Errors](errors.md)）
- capability manifest: [`../specs/0025`](../specs/0025-capability-manifest.md)
- embedder 定義 capability（`host.*`）: [`../specs/0026`](../specs/0026-embedder-defined-capabilities.md)
