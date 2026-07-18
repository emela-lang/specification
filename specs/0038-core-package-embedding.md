## 0038: Core Package Embedding and the core/std Boundary

Status: Draft

コンパイラに同梱される **embedded core** と，外部パッケージとして配布される **std** の境界を定める仕様．`intrinsic fn`（0021）／標準 registry の `extern fn`（0013）を宣言するモジュールを embedded core に置き，コンパイラ本体とバージョン一体で供給する．0021 の Open Question「Core Prelude の供給方法」を「ソース埋め込み」で解決し，その方式を Core Prelude 以外の標準モジュールへ拡張する．

### Summary

- コンパイラは **embedded core モジュール集合** をバイナリに同梱する．本仕様時点の集合は `core` / `io` / `clock` / `string` / `float` である．
- `std.core`（Core Prelude）は従来どおりすべての compilation unit に暗黙 import される（0021）．その他の embedded モジュールは明示 import（`import std.io` 等）で解決され，`--package` の供給を要しない．すべてのフロントエンド（check / build / run / LSP / embedder API）で一様に解決される．
- **境界ルール**：`intrinsic fn`，および標準 registry（0013）の `extern fn` を宣言するモジュールは embedded core に属する．ユーザーソース・パッケージでの `intrinsic fn` 宣言はエラーである．
- embedded モジュール名は `std` パッケージ名前空間で**予約**される．外部パッケージが `std` として同名モジュールを供給した場合，import の有無によらずハードエラーとする（サイレントシャドウ禁止）．
- 同名 intrinsic の二重宣言はエラーである（従来実装の silent dedup を撤去する）．
- 帰結として，外部 stdlib パッケージ（`std` Pome）は **純 Emela モジュールのみ**（`list` / `option` / `result` / `ord` / `int` 等）を供給する．contribute の受け口はこの層である．

### Motivation

現状，標準ライブラリ相当のコードは3箇所に分裂している：コンパイラ埋め込みの Core Prelude（`core.emel`），外部 stdlib パッケージ，テストフィクスチャ．このうち `intrinsic fn` / `extern fn` を宣言するモジュールは，コンパイラ・backend と厳密なバージョンロックを要する．

1. intrinsic 名は backend の命令表（0021）と一致しなければならず，extern の canonical 名は backend の供給実装（0013）と一致しなければならない．これらが外部パッケージにあると，コンパイラとパッケージのバージョンスキューが「型検査は通るが codegen で未知 intrinsic」という壊れ方を生む．
2. intrinsic の追加は必ずコンパイラ（命令表）と宣言（.emel）の同時変更になる．宣言が別リポジトリにあると，アトミックな変更が原理的に不可能になる（2 PR の順序調整が常態化する）．
3. 現行実装は，埋め込み prelude と外部 stdlib が同じ intrinsic を宣言しうるため，重複宣言を暗黙に無視している．境界を定めればこの緩和は不要になり，重複を正しくエラーにできる．

他言語の前例：Flix は stdlib 全体をコンパイラ jar に同梱し毎回ソースから読み込む．Scala は stdlib をコンパイラと同一リポジトリで管理する．Gleam は stdlib を完全に独立したパッケージにするが，これは演算子・等価比較が言語組み込みで stdlib がコンパイラの意味論に食い込まないから成立している——演算子の意味を stdlib へ移した Emela（0020/0021）では同じ前提が成り立たない．よって「コンパイラ結合層は同梱，純 Emela 層は外部パッケージ」の二層が Emela の設計に整合する分割である．

### Specification

以下，キーワードは RFC 2119 に従う．

#### embedded core モジュール

- コンパイラは embedded core モジュールのソースをバイナリに同梱する（MUST）．本仕様時点の集合は次のとおりである：

```text
std.core     -- Core Prelude（0021）: 演算子 trait とその組み込み instance，算術・比較・連結の intrinsic
std.io       -- effect io（0036）: write_stdout / write_stderr の extern と print / eprint
std.clock    -- effect clock: monotonic_seconds の extern と now
std.string   -- string_length / string_char_at / string_slice の intrinsic と安全なラッパー（0030）
std.float    -- f64_sqrt の intrinsic と数値ヘルパー
```

- 将来のモジュール追加（`fs`, `random` 等）は本仕様を supersede せず拡張する．追加の時期・内容は各機能仕様が定める．
- `std.core` は従来どおりすべての compilation unit に暗黙 import される（MUST，0021 不変）．
- その他の embedded モジュールは，明示 import によって解決される（MUST）．解決に `--package` その他の外部供給を要してはならない（MUST NOT）．
- embedded モジュールの解決は，すべてのフロントエンド（CLI の check / build / run，LSP，embedder API）で一様でなければならない（MUST）．
- import の形式・可視性・effect 操作の規則（0010 / 0018 / 0036）は，embedded モジュールに対しても通常のモジュールと同一である（MUST）．per-item import（`import std.string.length`）も通常規則どおり有効である．

#### 境界ルール

- `intrinsic fn` を宣言できるのは embedded core のソースのみである（MUST）．ユーザーソースまたはパッケージ内の `intrinsic fn` 宣言はコンパイルエラーである（MUST）．診断は宣言ごとに報告し，コンパイルは残りの検査を継続する（SHOULD，0033 のエラー収集方針）．
- `extern fn` は従来どおり 0013 の registry 検査に従う（不変）．標準 registry のエントリを宣言する標準モジュールは embedded core に置く．embedder による `host.*` の拡張（0026）は本ルールの対象外であり，host interface package が extern を宣言する唯一の経路として別仕様が定める．
- 以上の帰結として，「`intrinsic fn` / 標準 `extern fn` を宣言するモジュールは embedded core に属する」が core / std の境界の判定規則である．外部 stdlib パッケージは純 Emela モジュールのみを供給する．

#### 予約名

- `std` パッケージ名前空間の直下のモジュール名のうち，embedded core のモジュール名（本仕様時点：`core`, `clock`, `float`, `io`, `string`）は予約される（MUST）．
- `--package` または依存 Pome（0032）が，パッケージ名 `std` で予約名と同名のモジュールを供給する場合，コンパイルはエラーにしなければならない（MUST）．このエラーは当該モジュールが import されるか否かによらず検出する（MUST）．診断は衝突したモジュールと，embedded であることを明示すべきである（SHOULD）．
- `std` 以外のパッケージ名で同名のモジュール（例：`mylib.io`）を供給することは合法である．

#### 単一宣言ルール

- 同名の intrinsic を二重に宣言することはエラーである（MUST）．（従来実装は埋め込み prelude と外部 stdlib の重複を暗黙に無視していた．境界ルールにより embedded core が唯一の宣言箇所となるため，この緩和を撤去する．）

### Examples

`--package` なしで io が使える：

```emela
import std.io

fn main() -> Unit uses { io } {
    io.print("Hello, Emela!\n")
}
```

ユーザーソースの intrinsic はエラー：

```emela
intrinsic fn i32_add(a: Int, b: Int) -> Int uses {}
-- error: `intrinsic fn i32_add` may only be declared by the compiler's embedded core (spec 0038)
```

外部パッケージ `std` が予約名を供給するとエラー（import の有無によらない）：

```text
$ emela check main.emel --package ./my-std    # my-std が src/io.emel を含む
error: package `std` at `./my-std` provides `io.emel`, but module `std.io` is
embedded in the compiler (spec 0038); remove the module or rename the package
```

### Compilation Notes

この節は非規範的な実装上の補足である．

- **同梱方式**: embedded モジュールはコンパイラソースに `.emel` ファイルとして置き，`include_str!` で埋め込む（現行の Core Prelude と同方式）．コンパイル時に毎回パースする（Flix と同方式）．モジュール数が小さいためコストは無視できる．LSP など長命プロセスはパース結果をキャッシュしてよい．
- **解決機構**: import 解決で，パッケージ探索・相対パス解決より**前**に embedded 集合を照合する．モジュールのソース位置は実ファイルを持たないため，仮想パス（例 `<std.io>`，`<core-prelude>` の慣例に合わせる）で表し，パスの正規化（canonicalize）・overlay・ファイル読み込みをスキップする．これによりファイルシステムを持たない環境（wasm playground）でも解決できる．
- **診断**: embedded モジュール内に span を持つ診断は，開けるファイルが存在しない．LSP は entry ドキュメントへの要約診断として報告する．将来 go-to-definition を実装する際は，read-only の仮想ドキュメントとして embedded ソースを供給する必要がある（0033 の将来課題）．
- **予約名検査**: パッケージロード時（import 解決前）に，`std` パッケージのソースルート直下に予約名のモジュールファイルが存在するかを検査する．eager に検査することが「サイレントシャドウ禁止」の実装である．
- **外部 stdlib の移行**: 本仕様の適用に伴い，外部 stdlib パッケージから `io` / `clock` / `string` / `float` を削除する．旧コンパイラの利用者が `std.io` を失わないよう，コンパイラのリリース（embedded 供給開始）の後に stdlib パッケージの新バージョンをタグ付けする．

### 他仕様との関係

- **0021（Intrinsic Functions）**: Open Question「Core Prelude モジュールの正式名とコンパイラへの供給方法（ソース埋め込み vs `--package` 供給）」を「ソース埋め込み」で解決する．Core Prelude の暗黙 import の規範は不変．
- **0013（Platform Functions）**: Open Question「`extern fn` をアプリケーションでも宣言可能にするか，`std` パッケージに限定するか」を部分解決する——標準 registry の extern 宣言は embedded core に置く．registry の内容・カバレッジ検査は不変．
- **0026（Embedder-Defined Capabilities）**: host interface package は，embedded core の外で `extern fn` を宣言する唯一の経路である（境界ルールの明示的例外，0026 が規定）．
- **0032（Packaging）**: 外部 stdlib Pome は純 Emela 層のみを供給する．予約名検査は依存 Pome にも適用される．
- **0033（Language Server）**: embedded モジュールの仮想パスとその診断の扱い（Compilation Notes 参照）．
- **0036（Effect Declarations）**: embedded の `io` / `clock` は 0036 の effect 宣言のままである．0037（Module-Unit Imports and First-Class Effects，Draft）を将来採用する場合，embedded core の改名・書き換えはコンパイラリポジトリ内で完結する（本仕様が同梱を定めることの副次効果）．

### Open Questions

- embedded 集合の拡張基準の明文化（どの機能仕様が何を根拠にモジュールを追加するか；`fs` / `random` / `net` の導入仕様との接続）．
- 外部プラグイン backend（0013 Compilation Notes の external backend）向けに，intrinsic 宣言の escape hatch を許すか（現時点では不要と判断し，許さない）．
- stdlib Pome が要求する最低コンパイラバージョンの表明方法（Pome.toml の `emela` フィールドとの関係，0032）．
- LSP go-to-definition のための embedded ソースの仮想ドキュメント供給（0033）．
