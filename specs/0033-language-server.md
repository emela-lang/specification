# 0033: Language Server

Status: Draft

Emela の Language Server Protocol（LSP）サーバ `emela lsp` と，その前提となるコンパイラフロントエンドの
**複数エラー収集**を定義する仕様．エディタ（Neovim / VSCode 等）から編集中の Emela ソースに対して，
コンパイラが発するすべての診断（字句・構文・import・型・trait 実装漏れ・effect・throws・entrypoint）と，
文脈に応じた補完（import 候補，`match` / `catch` の enum バリアント，`uses` 行の effect，予約語ほか）を提供する．

## Summary

- CLI に `emela lsp [--package DIR ...]` を追加する．サーバは **stdio** 上で JSON-RPC 2.0 を話す
  （LSP Base Protocol，`Content-Length` ヘッダによるフレーミング）．
- 位置エンコーディングは LSP 既定の **UTF-16** とする．
- 診断は didOpen / didChange / didSave のたびにフロントエンド全段
  （lex → parse → import 解決 → prelude マージ → 型検査）を実行して配信する．
  コンパイラが出しうる**すべてのエラー種別**が診断として現れる．
- フロントエンドは**複数エラーを収集**する．各ステージ・各宣言の境界でエラーを回収して継続し，
  1 回の検査で独立なエラーを同時に報告する（詳細は「複数エラー収集」）．これは `emela check` の
  CLI 出力にも適用され，全エラーが空行区切りで出力される．
- 補完は 6 つの文脈を判別して候補を返す: import パス，`::` 直後のバリアント/組み込み変換，
  `uses { ... }` 内の effect 名，`match` アームのバリアント，`catch` アームのエラーバリアント，
  および既定文脈（予約語・型名・スコープ内関数・ローカル束縛）．
- 未保存バッファの内容は **overlay** として import 解決に反映される．開いているファイルはディスクではなく
  エディタの最新内容で解決される．
- プロトコル実装は外部クレートに依存せず，serde / serde_json のみで自前実装する（0032 の CLI と同方針）．

## Motivation

Emela には現在エディタ支援が存在せず，エラーの発見は `emela check` の手動実行に限られる．また現行の
フロントエンドは最初のエラーで停止するため，修正すべき箇所を一度に把握できない．言語の核である
「effect row = 権限表」（0000 / 0009）や `throws E`（0011），trait（0020）は宣言的な注釈を要求する設計で
あり，注釈の候補提示と即時の診断はエディタ上でこそ価値が大きい．

LSP を標準プロトコルとして採用すれば，単一のサーバ実装で Neovim / VSCode ほか任意のクライアントに対応
できる．依存を増やさない自前実装は，コンパイラ本体と同じ配布物（単一バイナリ）にサーバを同梱すること
を容易にする．

## Specification

### 起動と転送路

```
emela lsp [--package DIR ...]
```

- サーバは stdin から要求・通知を読み，stdout に応答・通知を書く．stderr はログ専用とする．
- フレーミングは LSP Base Protocol に従う（`Content-Length: N\r\n\r\n` + JSON 本文）．
- `--package` は `check` / `build` と同じ意味を持つ（0032）．加えて，開かれた各ファイルについて
  そのファイルを含むプロジェクトの依存 Pome（`Pome.toml` / `Pome.lock`）を 0032 M1 の規則で解決し，
  import root として合成する．すなわち診断・補完の package 解決は `emela check FILE` と一致する．
- 要求の処理は単一スレッドで逐次に行ってよい（性能目標: 通常規模のファイルで編集ごとの全段検査が
  対話速度で完了すること）．

### ライフサイクルとメソッド

サーバが処理するメソッドは以下に限る．未知の要求には `MethodNotFound`（-32601）を返し，未知の通知は
無視する．

| 種別 | メソッド | 挙動 |
| --- | --- | --- |
| 要求 | `initialize` | capabilities（下記）と serverInfo を返す |
| 通知 | `initialized` | 何もしない |
| 通知 | `textDocument/didOpen` | 文書を登録し検査・診断配信 |
| 通知 | `textDocument/didChange` | 全文同期（`TextDocumentSyncKind.Full`）で更新し検査・診断配信 |
| 通知 | `textDocument/didSave` | 検査・診断配信 |
| 通知 | `textDocument/didClose` | 文書を破棄し，その URI の診断を空配列でクリア |
| 要求 | `textDocument/completion` | 文脈判別して補完候補を返す |
| 要求 | `shutdown` | `null` を返し終了待機状態に入る |
| 通知 | `exit` | `shutdown` 受領済みなら終了コード 0，未受領なら 1 で終了 |
| 通知 | `$/cancelRequest` | 無視する（処理は逐次のため常に完了済み） |

- `initialize` 前に届いた要求には `ServerNotInitialized`（-32002）を返す．
- capabilities:
  - `textDocumentSync`: `{ openClose: true, change: Full, save: true }`
  - `completionProvider`: `{ triggerCharacters: [".", ":", "{"] }`
  - `positionEncoding`: `"utf-16"`

### 複数エラー収集

フロントエンドは「最初のエラーで停止」をやめ，**宣言・ステージ境界で回収して継続**する．
式の検査（関数本体内部）は従来どおり最初のエラーで打ち切る．粒度は次のとおり:

- **字句解析**: 不正な文字はエラーを記録してその文字を読み飛ばす．不正なリテラル（`Int` の桁あふれ等）は
  エラーを記録し，プレースホルダのトークンを産出して継続する．
- **構文解析**: top-level 宣言のパースに失敗したら，エラーを記録し，ブレース深さ 0 で行頭に現れる次の
  宣言開始トークン（`fn` / `pub` / `enum` / `trait` / `impl` / `extern` / `intrinsic` / `import` /
  `module`）まで読み飛ばして再開する．成功した宣言だけからなる部分的な `Program` を後段に渡す．
- **import 解決**: import 文 1 本ごとにエラーを回収し，残りの import を処理する．
- **型検査**: 宣言の登録（enum / trait / impl / fn / extern）は項目ごとに回収し，登録できたものだけで
  表を構築して続行する．関数本体・impl メソッド本体はそれぞれ独立に検査し，1 本体につき最初の 1 件を
  報告する．
- **重複排除**: 部分的な登録に起因する連鎖エラーを抑えるため，収集後に（タイトル，span）が同一の診断を
  1 件に併合する．
- エラーが 1 件でもあれば検査は失敗である（部分的な `Program` から後段の lowering へ進むことはない）．

`emela check` はこの収集結果を**空行区切りで全件出力**し，1 件でもあれば終了コード 1 とする．
`build` / `ir` も全件を報告するが，エラーがある限り成果物を生成しない．文字列 API（playground 向け
`check_source` 等）の既存シグネチャは互換のため維持し，先頭 1 件を返す．

### 診断

- didOpen / didChange / didSave のたびに，当該文書を入口としてフロントエンド全段を実行する．
- **entrypoint 規則**: 検査対象のファイル自身が `fn main` を宣言している場合に限り `main` の存在・形状
  （0003）を検査する．宣言していないファイルは `check --library` と同様に扱い，`Missing entrypoint` を
  発しない．
- **overlay**: サーバが開いている文書の最新内容は，import 解決がディスクを読む前に参照される．
  未保存の編集が依存側・被依存側の双方の診断に反映される．
- 変換規則: 各エラーの `Diagnostic` から LSP `Diagnostic` を作る．
  - range: 主ラベルの span（バイトオフセット）を UTF-16 の行・列に変換する．span を持たないエラー
    （IO 失敗，循環 import 等）はファイル先頭の `0:0..0:0` とする．
  - message: タイトルにラベルメッセージと Hint を連結した可読文字列．
  - severity: Error（現行コンパイラに警告は存在しない）．source: `"emela"`．
- **配信先**: 診断は span が指すファイルの URI へ配信する．入口文書以外（import されたモジュール）に
  span があるエラーは，そのファイルの URI に配信し，あわせて入口文書にも「import 先のエラー」である旨の
  要約診断を 1 件配信する（入口文書のどこが原因か視認できるようにするため）．
- 検査のたびに，前回その検査で診断を配信した URI のうち今回エラーがなくなったものへ空配列を配信して
  クリアする．

これによりコンパイラが発しうるエラーは**定義上すべて**診断として現れる: エラーは全経路が単一の
`Error` 型（`Diagnostic` 付き）に集約されているため，新しいエラーを追加しても LSP 側の変更なしに
診断へ反映される．

### 補完

補完はカーソルまでのトークン列（コンパイラの lexer を再利用）を後方走査して文脈を判別する．
候補の情報源は**スコープスナップショット**である: parse・import 展開・prelude マージまでで得られた
`Program` から，enum（バリアントとフィールド数），trait，公開関数（シグネチャ），extern，effect 名集合を
抽出したもの．検査が失敗しても構文回復により部分スナップショットが得られ，全滅時は直近の成功時の
スナップショットを用いる．

判別する文脈と候補（上から順に判定し，最初に該当したものを用いる）:

1. **import 行**（カーソル行が `import` で始まる）: ドットで区切られた入力済みプレフィックスに応じて，
   - 先頭セグメント: package 名（`--package` + 依存 Pome の import root）と，当該ファイルと同階層の
     `.emel` モジュール名，
   - 中間セグメント: package の source root 配下のディレクトリ / `.emel` ファイル名，
   - 末尾セグメント: 解決先モジュールの `pub fn` 名．
2. **`::` 直後**: 直前の識別子が enum 名ならそのバリアント．payload を持つバリアントは
   スニペット（例 `Some(${1:value})`）で挿入する．`Char` / `String` には組み込み変換
   `Char::from_code` / `String::from_char`（0017）を提示する．
3. **`uses { ... }` 内**（後方に未閉の `{` があり，その直前が `uses`）: スナップショット中の全宣言の
   effect row に現れる effect 名．
4. **`match` アーム位置**（未閉の `{` の直前が `match <scrutinee>`）: scrutinee が単純識別子で，囲み関数の
   引数か型注釈付き `let` から enum 型が特定できればその enum のバリアントを優先する．特定できなければ
   スコープ内の全 enum のバリアント．挿入形は `Variant` / `Enum::Variant` の両方（0005 / 0018 R7）．
5. **`catch` アーム位置**（未閉の `{` の直前が `catch`）: スコープ内で `throws` 節に現れる enum の
   バリアントを優先し，フォールバックとして全 enum のバリアント．
6. **既定**: 予約語（`fn` `extern` `intrinsic` `trait` `impl` `for` `import` `let` `module` `pub` `uses`
   `enum` `match` `if` `else` `throws` `throw` `try` `catch` `panic` `true` `false`），
   型名（`Unit` `Bool` `Int` `Float` `String` `Char` `Array` `Option` `Self`），スコープ内の関数名
   （qualified 形を含む，0018）・enum 名・trait メソッド名，囲み関数のパラメータと `let` 束縛．

補完候補の `detail` には関数シグネチャや所属 enum を示す．文脈判別に失敗した場合は既定文脈（6）に
フォールバックする．

### エディタ統合

- リポジトリは最小の VSCode 拡張（`editors/vscode/`: 言語登録 `.emel`，`emela lsp` の起動，行コメント
  `--` 等の言語設定，簡易ハイライト）と，Neovim（`vim.lsp.config` / nvim-lspconfig）の設定例
  （`docs/lsp.md`）を同梱する．root marker は `Pome.toml` を第一候補とする．

## Examples

複数エラーの同時報告（2 つの独立に壊れた関数）:

```
fn f() -> Int uses {} {
  "text"            -- Type mismatch（1 件目）
}

fn g() -> Int uses {} {
  unknown_name      -- Unknown name（2 件目）
}
```

`emela check` は両方を空行区切りで出力し，LSP は同一ファイルに 2 つの診断を配信する．

`catch` 補完:

```
fn parse_or(s: String, fallback: Int) -> Int uses {} {
    try {
        parse_digit(s)
    } catch {
        ParseError::│        -- ここで Empty / BadDigit を提示
    }
}
```

`uses` 補完:

```
fn main() -> Unit uses { │ }   -- ここで io / clock など，スコープ内の effect 名を提示
```

## Compilation Notes

- サーバは `crates/emela` 内のモジュールとして実装する（フロントエンド内部型に `pub(crate)` で触れる
  ため）．プロトコル層は serde / serde_json のみで実装し，新規依存を持ち込まない．
- 複数エラー収集は各ステージの戻り値を「値 + `Vec<Error>`」へ広げることで実現する．式レベルの
  `Result` 配管は変更しない．
- `Span` はバイトオフセットであり，`SourceFile` がソース全文を保持するため，UTF-16 変換に際して
  ファイルの再読は不要である．

## Open Questions

- 1 本体内の複数エラー（式レベルの収集）をどこまで進めるか．
- hover / go-to-definition / rename / フォーマッタ等の追加機能．
- didChange のデバウンスと incremental sync が必要になる規模はどこか．
- workspace 全体（開いていないファイル）への診断配信を行うか．
