## 0040: Unit Testing — `@test` and `emela test`

Status: Draft

テスト名は `モジュール名.関数名` に固定し，`@test` 本体は暗黙 try とする（2026-07-18 決定）．

単体テストの宣言と実行を定める仕様．属性 `@test`（0039）でトップレベル `fn` をテストと宣言し，0032 の C2 で予約された `emela test` が現在の Pome のテストを発見・コンパイル・実行する．テストは本体が正常に return すれば成功，未捕捉の error（0011）・`panic`・runtime trap で失敗する．エントリポイントはコンパイラが合成するハーネスであり，`main` の `throws Never` 規則（0011）は不変である．アサーションはコンパイラ特権を持たない通常のライブラリ（標準は `github.com/emela-lang/brix`）とし，開発時にのみ要る依存のために `Pome.toml` へ `[dev-dependencies]` を追加する（0032 拡張）．

### Summary

- `@test` はトップレベル `fn` に付ける．テスト関数はパラメータ 0 個・戻り値 `Unit`・`throws` 節なし・型パラメータなし．`uses` は自由に宣言できる．
- テストはプロダクションコードと同じモジュールファイルに同居できる．同一コンパイル単位の非公開関数をベア名でテストできる．
- **暗黙 try**：テスト関数の本体では throwing な呼び出しと `throw` をベアで書ける．送出された error はその地点でテスト失敗としてハーネスへ伝播する．0011 の「`?` か try / catch」規則に対する唯一の例外である．
- `emela test` は現在の Pome のソースルート配下の全モジュールからテストを発見し，逐次実行して結果を報告する．依存 Pome・embedded std（0038）のテストは実行しない．
- `check` / lint / LSP はテスト関数を型検査する．`build` / `run` / `ir` はテスト関数を成果物から除外する．
- `Pome.toml` に `[dev-dependencies]` を追加する．dev 依存は現在の Pome のフロントエンドで解決に参加するが，成果物に到達したらエラーとする．依存として使われる Pome の dev 依存は解決されない．
- ランナーとアサーションライブラリの契約は「throw = 失敗」だけである．標準アサーションライブラリ brix は通常の library Pome である．

### Motivation

1. **Emela コードを Emela のテストで検証する手段がない**．現状，stdlib（0038 の純 Emela 層）の検証はコンパイラリポジトリの Rust 統合テスト（`emela run` の stdout 比較）に依存し，ライブラリ作者は自分のリポジトリでテストを書けない．0038 が定めた contribute の受け口にテスト基盤がないことは，エコシステムのボトルネックである．
2. **同居テスト**．単体テストの第一の対象は，モジュールの非公開関数を含む「単位」である．テストを別ファイル・別ディレクトリに限ると pub API しか見えず（0037 R5），可視性を緩める別機構が要る．宣言と同じファイルに置ければ，可視性規則は不変のまま非公開関数をベア名でテストできる（Rust の `mod tests` / Zig の `test` ブロックと同じ判断）．
3. **ランナーはコンパイラが持つ**．`emela test` は 0032 の C2 で予約済みである．発見・実行・報告をツールチェーンの標準にしてフレームワークの分裂を避ける．エントリをコンパイラ合成のハーネスにすることで，`main` の `throws Never`（0011）・可視性（0010）の規則に例外を作らずに済む．
4. **失敗 = throw**．テストの失敗表現に新しいチャネルを導入しない．アサーションは「失敗したら throw する通常の関数」（0011）であり，ランナーは error 型もライブラリも特別扱いしない．これによりアサーションライブラリは通常の Pome として配布・代替できる．
5. **暗黙 try**．0011 をそのまま適用すると，すべてのアサーション呼び出しに `?` が要り，テスト関数は `throws` の宣言と error 型の合流（0011「error 型の一致」）を強いられる．テスト本体では「error はその場で失敗」が常に望む意味論なので，本体に限り伝播を暗黙にする．

### Specification

以下，キーワードは RFC 2119 に従う．

#### `@test` 宣言（T 規則）

- **T1（適用対象）**: `@test` はトップレベルの `fn` 宣言にのみ適用できる（MUST）．それ以外の宣言種・trait / impl のメソッド・effect ブロック内の操作への適用はエラーである（0039 R6）．
- **T2（シグネチャ）**: `@test` を付けた関数（以下テスト関数）は，パラメータを持ってはならず（MUST NOT），戻り値型は `Unit` でなければならず（MUST），`throws` 節を宣言してはならず（MUST NOT），型パラメータを持ってはならない（MUST NOT）．
  - 戻り値を `Unit` に固定するのは，合否が「正常に return したか」だけで決まり，戻り値に消費者がいないためである．`throws` の禁止は T3 の暗黙チャネルによる（宣言する対象がない）．型パラメータの禁止は，テスト関数に呼び出し箇所がなく（T5），単相化の実引数が決まらないためである．
- **T3（暗黙 try）**: テスト関数の本体の直下（入れ子の関数リテラルの本体を除く）では，throwing な呼び出しと `throw` 式をベアで書いてよい（MAY）．これは 0011 の「throwing な呼び出しは `?` を付けるか `try` block 内に置く」規則に対する唯一の例外である．送出された error はその地点でテスト失敗としてハーネスへ伝播し，本体の残りは評価されない（短絡）．
  - 伝播先はハーネスの受け口であり，通常の `throws` チャネルではない．送出サイトごとに error 型が異なってよく，型の合流・変換（0011「error 型の一致」）を要しない．
  - 本体内の明示的な `try { } catch { }` は 0011 どおりに機能する（error 経路のテストに用いる）．`catch` されなかった error（`catch` arm 内での再 `throw` を含む）は暗黙チャネルへ伝播する．
  - `?` は 0011 どおり「囲む関数の宣言 `throws` への伝播」であり，テスト関数は `throws` を持たない（T2）ため使えない（エラー）．診断はベア呼び出しで足りることを案内すべきである（SHOULD）．
  - 入れ子の関数リテラルの本体は通常の規則（0011）に従う．
- **T4（effect）**: テスト関数は任意の effect row を `uses` に宣言してよい（MAY）．`emela test` は `emela run` と同一の標準 platform（0013）を供給しなければならない（MUST）．供給されない capability の要求は，通常どおりカバレッジ検査（0013 / 0025）のエラーである．
- **T5（可視性と参照禁止）**: テスト関数に `pub` を付けてはならない（MUST NOT）．テスト関数は，他のテスト関数を含むいかなるソースコードからも，ベア名・修飾名を問わず参照できない（MUST NOT）．テスト関数は通常ビルドの成果物から除外される（T8）ため，参照を許すとテストモードと通常ビルドで名前解決が分かれてしまう．
- **T6（テスト ID）**: テストの識別子は「ソースルートからのモジュールパス + `.` + 関数名」である（例：`list.append_keeps_order`）．同一モジュール内の関数名の重複は既存の規則でエラーになるため，ID は Pome 内で一意である．文字列等による表示名の上書きは持たない．
- **T7（配置）**: テスト関数は現在の Pome の任意のモジュールに，通常の宣言と並べて置ける（MAY）．テスト本体からの参照は通常の名前解決（0037）に従う：同一コンパイル単位の非公開関数はベア名で呼べるため，モジュールの内部関数を直接テストできる（T5 が禁じるのはテスト関数**への**参照のみである）．
- **T8（コンパイルモード）**: `emela check`・`emela lint`・LSP（0033）は，テスト関数を通常の関数として型検査・effect 検査しなければならない（MUST）．`emela build` / `run` / `ir` はテスト関数を成果物から除外しなければならない（MUST）：lowering・emit されず，capability manifest（0025）に寄与しない（MUST NOT）．`build` / `run` の `main` 要求は不変である．

#### ランナー（C 規則）

- **C1（対象）**: `emela test` は，カレントディレクトリから上方に最も近い `Pome.toml` を持つ Pome を対象とする（MUST）（0032 C2 の具体化）．`Pome.toml` が見つからない場合はエラーである．
- **C2（発見）**: ランナーは現在の Pome のソースルート（規約 `src/`）を再帰走査し（`.` で始まるディレクトリは無視，0035 の fmt CLI と同じ規約），すべての `*.emel` モジュールのテスト関数を発見する（MUST）．発見は import 到達性に依存しない：どこからも import されないモジュールのテストも対象である．依存 Pome と embedded std モジュール（0038）のテストを発見・実行してはならない（MUST NOT）．列挙順はモジュールパスの辞書順，同一モジュール内は宣言順とする（SHOULD）．
- **C3（コンパイルとエントリ）**: 発見された各モジュールは通常のフロントエンド（0037 の解決規則・0038 の embedded 解決を含む）でコンパイルされる（MUST）．`main` は要求されない（MUST NOT）：存在してもエントリポイントにはならない．エントリはコンパイラが合成するハーネスであり，各テスト関数を直接呼び出す．合成ハーネスはソースコードではないため，可視性（0010）と T5 の参照禁止の対象外である．backend は `emela run` と同一である．
- **C4（合否）**: テストは，本体が正常に return したとき，かつそのときに限り成功である（MUST）．暗黙チャネル（T3）へ error が伝播した場合・`panic`（0011）した場合・runtime trap した場合は失敗である（MUST）．
- **C5（隔離と順序）**: あるテストの失敗（trap を含む）は，他のテストの実行と結果に影響してはならない（MUST）．本仕様のランナーは逐次実行である．テストは実行順序に依存してはならない：順序は保証されない（C2 の列挙順は報告の順序であって実行意味論の保証ではない）．
- **C6（報告）**: ランナーは（1）実行開始時にテスト総数，（2）テストごとに ID と `ok` / `FAILED`，（3）失敗したテストの詳細，（4）末尾に合計を報告しなければならない（MUST）．形式は次に従うべきである（SHOULD）：

  ```text
  running 3 tests
  test rle.empty_input_has_zero_runs ... ok
  test rle.repeated_chars_collapse ... FAILED
  test rle.rejects_bad_digit ... ok

  failures:

  ---- rle.repeated_chars_collapse ----
  threw AssertError: assertion failed: expected `2`, got `3`

  test result: FAILED. 2 passed; 1 failed
  ```

- **C7（失敗の描画）**: 暗黙チャネルに伝播した error の型 `E` に対し，スコープ内に `impl Show for E`（0020）があれば `to_string` で値を描画する（SHOULD）．なければ型名のみを報告する．`E` は送出サイトごとに静的に既知であり（T3），実行時の型情報を要しない．`panic` はメッセージを報告する（SHOULD）．trap は種別を報告してよい（MAY）．
- **C8（終了コード）**: すべてのテストが成功（テスト 0 件を含む）なら 0，1 件でも失敗すれば 1 で終了する（MUST）．コンパイルエラーの場合は通常の診断を報告して非 0 で終了する（MUST）．
- **C9（成果物）**: `emela test` はファイルシステムに成果物を残さないことが望ましい（SHOULD）（`emela run` と同様の in-process 実行）．

#### 開発時依存（D 規則，0032 拡張）

- **D1（宣言）**: `Pome.toml` に `[dev-dependencies]` テーブルを追加する．形式は `[dependencies]`（0032 F3）と同一：canonical source path から version requirement への対応である．
- **D2（解決）**: dev 依存は通常の解決器（0032 V3）で `[dependencies]` と合わせて解決され，`Pome.lock` に記録される（MUST）．
- **D3（可視範囲）**: 現在の Pome のコンパイルでは，dev 依存の import ルート（0032 M2）は `check` / lint / LSP / `test` のすべてのフロントエンドで解決に参加する（MUST）．テスト関数を型検査する（T8）ために必要である．import ルート名の衝突は通常依存と同じ規則でエラーとする．
- **D4（成果物ガード）**: `build` / `run` / `ir` は，テスト関数の除外（T8）後の成果物から dev 依存のモジュールへ到達するコードがある場合，エラーとしなければならない（MUST）．`check` が同じ検査を報告してもよい（MAY）．これにより dev 依存が利用者の実行時依存へ昇格することはない．
- **D5（非推移性）**: Pome が依存として解決されるとき，その Pome の `[dev-dependencies]` は解決されない（MUST NOT）．依存のテストは実行されない（C2）ため不要である（cargo と同方針）．

#### アサーションライブラリとの契約（B 規則）

- **B1（契約は throw のみ）**: ランナーとアサーションライブラリの契約は C4 がすべてである．ランナーは特定のライブラリ・error 型・関数を特別扱いしてはならない（MUST NOT）．アサーションは「失敗したら throw する通常の関数」であり，ライブラリなしの素の `throw` だけでもテストは書ける．
- **B2（標準ライブラリ brix，informative）**: 標準のアサーションライブラリとして `github.com/emela-lang/brix` を emela-lang Org. に置く（名は果実の糖度・品質の測定単位 °Brix から．Pome / Bushel / Orchard の果樹メタファー（0032）に連なる）．import ルートは source path の leaf 既定で `brix`（0032 M2，`[pome].module` の上書き不要）．レイアウトは stdlib と同形（`Pome.toml` + `src/assert.emel` 等）の純 Emela Pome（0038）であり，コンパイラ特権を持たない．API は brix リポジトリが定め，本仕様は固定しない．参考として v1 の骨子を示す：

  ```emela
  module assert

  pub enum AssertError {
      Failed(String)
  }

  impl Show for AssertError {
      fn to_string(e: AssertError) -> String uses {} {
          match e {
              AssertError::Failed(message) -> "assertion failed: " ++ message
          }
      }
  }

  pub fn eq<T: Eq + Show>(actual: T, expected: T) -> Unit throws AssertError uses {}
  pub fn ne<T: Eq + Show>(actual: T, unexpected: T) -> Unit throws AssertError uses {}
  pub fn ok(condition: Bool) -> Unit throws AssertError uses {}
  pub fn fail(message: String) -> Unit throws AssertError uses {}
  ```

  `Eq` / `Show` は Core Prelude の trait（0020 / 0021）である．`eq` / `ne` は `T: Eq + Show` を要求し，失敗時に両辺を描画した message で throw する．`AssertError` が `Show` を実装しているため，ランナーの失敗報告（C7）に message がそのまま現れる．

### Examples

同居テストの基本形（暗黙 try により `?` は不要）：

```emela
-- src/rle.emel — プロダクションコードとテストが同じファイルに同居する
module rle

import brix.assert

fn run_length(s: String) -> Int uses {} {
    ...  -- 非公開のプロダクション関数
}

@test
fn empty_input_has_zero_runs() -> Unit uses {} {
    assert.eq(run_length(""), 0)      -- 失敗すれば throw され，その場でテスト失敗（T3）
}

@test
fn repeated_chars_collapse() -> Unit uses {} {
    assert.eq(run_length("aaabb"), 2)
}
```

effect を使うテスト（T4）——ランナーは `run` と同じ platform を供給する：

```emela
import std.io
import brix.assert

@test
fn boundary_case_with_debug_output() -> Unit uses { Io } {
    Io.print("debug: checking single-char input\n")
    assert.eq(run_length("a"), 1)
}
```

error 経路のテスト（T3）——本体内の明示 `try` / `catch` は通常どおり機能する：

```emela
-- parse_digit: fn (String) -> Int throws ParseError（同モジュールの非公開関数）

@test
fn rejects_bad_digit() -> Unit uses {} {
    let outcome = try {
        let _digit = parse_digit("x")
        "accepted"
    } catch {
        ParseError::BadDigit -> "rejected"
        e -> "other error"
    }
    assert.eq(outcome, "rejected")
}
```

`[dev-dependencies]` の宣言（D1）：

```toml
[pome]
name = "github.com/you/rle"
version = "0.1.0"
emela = "0.6"

[dev-dependencies]
"github.com/emela-lang/brix" = "^0.1"
```

実行（C6 / C8）：

```console
$ emela test
running 3 tests
test rle.empty_input_has_zero_runs ... ok
test rle.repeated_chars_collapse ... FAILED
test rle.boundary_case_with_debug_output ... ok

failures:

---- rle.repeated_chars_collapse ----
threw AssertError: assertion failed: expected `2`, got `3`

test result: FAILED. 2 passed; 1 failed

$ echo $?
1
```

シグネチャ違反はエラー（T2 / T5）：

```emela
@test
pub fn visible_test() -> Unit uses {} { () }
-- error: `@test` fn must not be `pub` (spec 0040)

@test
fn with_arg(x: Int) -> Unit uses {} { () }
-- error: `@test` fn takes no parameters (spec 0040)

@test
fn returns_int() -> Int uses {} { 42 }
-- error: `@test` fn must return `Unit` (spec 0040)
```

プロダクションコードから dev 依存へ到達するとビルドエラー（D4）：

```emela
-- src/main.emel（プロダクションコード）
import brix.assert

fn main() -> Unit uses {} {
    try { assert.ok(true) } catch { e -> () }
}
```

```text
$ emela build src/main.emel
error: module `assert` is provided by dev-dependency `github.com/emela-lang/brix`
and must not be reachable from build artifacts (spec 0040)
```

### Compilation Notes

この節は非規範的な実装上の補足である．

- **ハーネス合成**: 0011 の IR lowering（内部 Ok / Err 表現）をそのまま使える．暗黙 try（T3）は，本体直下の throwing 呼び出し・`throw` を「Err 分岐がハーネスの失敗報告へ分岐する」形へ lowering するだけで，`try` / `catch` の lowering と同型である．送出サイトごとに `E` が静的に既知なので，`impl Show for E` の有無に応じた描画コード（C7）を単相化時に選べる．
- **隔離**: wasm-wasi backend では trap 後の instance は再利用できないため，テストごとに fresh instantiate するのが自然な実装である（`emela run` の in-process 実行系——`proc_exit` / `fd_write` のみを link——を再利用できる）．全テストを 1 つの wasm モジュールへコンパイルし，テストごとの合成エントリを instance ごとに呼び分ける形を推奨する．
- **発見**: ソースルート走査は `emela fmt` のディレクトリ走査と同じ規約であり，実装を共有できる．
- **D3 / D4**: D3 は依存ロード（`Pome.lock` 由来の import root 合成）へ dev 依存を加える一点変更で満たせる．D4 は lowering の到達可能性計算（T8 のテスト除外後）で dev 依存 root 由来モジュールへの到達を検出する．この規則は「`@test` でない非公開ヘルパが brix を使う」ことを禁じない——ヘルパが成果物から到達不能である限り合法であり，テスト用ヘルパはこの形で書ける．

### 他仕様との関係

- **0039（Attributes）**: `test` を最初の認識属性として登録する（0039 R4）．適用対象はトップレベル `fn`（T1 が 0039 R6 を実体化する）．
- **0032（Packaging）**: C2 で予約された `emela test` の意味論を定める（本仕様 C1）．F3 を D 規則（`[dev-dependencies]`）で拡張し，`Pome.lock`（F5–F8）は dev 依存も記録する（D2）．依存の dev 依存は解決しない（D5）．
- **0011（Error handling）**: `throw` / `try`–`catch` / `?` の規則はテスト本体でも不変であり，T3 の暗黙 try が「throwing な呼び出しの置き場所」規則に対する唯一の例外である．`main` の `throws Never` 規則は不変（エントリは合成ハーネスであって `main` ではない）．`panic` はテスト失敗である（C4）．
- **0013 / 0025**: ランナーは `run` と同一の標準 platform を供給する（T4）．テスト関数は成果物の capability manifest に寄与しない（T8）．
- **0037（Module-Unit Imports and First-Class Effects）**: `import brix.assert` → `assert.eq(...)` は R1 / R2(a) の通常規則であり，本仕様は名前解決規則を追加しない（T5 の参照禁止を除く）．
- **0038（Core Package Embedding）**: テストから embedded std を import できる．embedded モジュール自身のテストは C2 の対象外である（コンパイラリポジトリの責務）．
- **0035（Canonical Formatting and Lint）**: 属性の整形は 0039 R8 が定める．lint はテスト関数も検査対象に含む：テストだけが使う import は `imports/unused` にならず（L1 の型検査にテストが含まれるため），`naming/snake-case` はテスト関数名にも適用される．
- **0033（Language Server）**: `check` 意味論にテストが含まれる（T8）ため，LSP 診断はテスト関数のエラーも報告する．エディタからのテスト実行（code lens / test explorer）は本仕様の範囲外である．
- **0010（Modules, Imports, Visibility）**: `pub` 禁止は T5．合成ハーネスは可視性規則の対象外である（C3）．

### Open Questions

1. 名前フィルタ（`emela test rle` / `emela test rle.empty_input_has_zero_runs`）と，Pome 外の単一ファイル指定．
2. 並列実行．C5 の隔離が前提を与える．報告順の安定化が論点．
3. 機械可読出力（JSON，CI 統合）．
4. 期待失敗テスト（`@test(expect_throw)` 等）——0039 R7 の引数文法の具体化を待つ．
5. テスト専用ディレクトリ（`tests/`）の規約——pub API だけを使う統合テスト層として．
6. setup / teardown・fixture——effect handler（0037 Open Question 4）の導入後に再訪する．capability の差し替えによる mock（0026）は現行機構で可能である．
7. Bushel（workspace，0032 F9 / F10）全体での `emela test`．
8. brix API の拡充（`Float` の近似比較・コレクション比較等）はライブラリ側の課題であり，本仕様の範囲外である．
