## 0043: Fallible Platform Functions — extern fn と throws

Status: Draft

platform 関数の失敗は `throws` チャネルで表す（2026-07-19 決定）．

platform 関数（0013）が失敗を報告するための規約を定める仕様．registry のエントリと `extern fn` 宣言に `throws E` を許し，ホストが報告する失敗を 0011 の error channel へ接続する．0013 が「失敗を返す platform 関数は後続仕様において追加する」と先送りした点を埋める．最初の適用例は HTTP クライアント（0044）である．

### Summary

- registry（0013）のエントリは `throws E` を宣言してよい（MAY）．宣言しないエントリは従来どおり失敗しない（`throws Never` と等価，0011）．
- ホストが報告する失敗は，呼び出し側から見て通常の Emela error である：throwing な呼び出しの規則（0011——`?` を付けるか `try` block 内に置く）がそのまま適用される．新しい構文・型は導入しない．
- `E` は当該 extern を宣言する embedded core モジュール（0038）内で宣言された，型パラメータを持たない `pub enum` である．
- 0013 の「失敗を返す platform 関数は `Result` / `Option` を戻り値とする形で後続仕様において追加する」という文言は，本仕様が `throws` 採用で置き換える（0011 が `Result` を組み込み型としないと決めたため）．
- error は capability ではない（0009）：`throws` の追加は `uses` row・カバレッジ検査（0013）・capability manifest（0025）に一切影響しない．
- 既存の `io.*` / `clock.*` エントリは非可謬のまま不変である．

### Motivation

1. **0013 の明示的な先送り**．現行 registry は「戻り値が `Unit` で，失敗を値で返さない最小集合」に限られ，失敗が本質的な操作（ネットワーク・ファイルシステム等）を追加できない．HTTP（0044 / 0046）は失敗する最初の標準 capability であり，失敗規約の確定がその前提条件になる．
2. **チャネルの一貫性**．0013 は失敗を「`Result` / `Option` 戻り」と書いたが，その後の 0011 は回復可能な失敗の表現を `throws E` に一本化し，`Result` を組み込み型としないと決めた．platform 関数だけが `Result` 風の戻り値を持つのは言語の error model と乖離する．ホストの失敗も Emela の失敗も同じ `throws` チャネルに流せば，呼び出し側の規則（`?` / `try`–`catch`・exhaustiveness）が一様に適用される．
3. **ABI は既にある**．`throws E` を持つ関数は IR 上，成功値と error の tagged union へ lowering される（0011 IR Lowering）．extern の失敗報告はこの表現をホスト境界まで延長するだけであり，新しい実行時機構を要しない．

### Specification

以下，キーワードは RFC 2119 に従う．

#### registry と extern 宣言（F 規則）

- **F1（エントリ形式）**: registry のエントリは `throws E` を持ってよい（MAY）．形式は次のとおり拡張される：

  ```text
  修飾名 : (引数型, …) -> 戻り値型 [throws E] uses { Capability }
  ```

  `throws` を持たないエントリの意味は従来どおりである（失敗しない；`throws Never` と等価，0011）（MUST）．
- **F2（error 型）**: `E` は，当該エントリの extern を宣言する embedded core モジュール（0038）内で宣言された，型パラメータを持たない `pub enum` でなければならない（MUST）．variant の payload 型は registry で表現可能な型に限る（MUST）．importer は 0037 R2(b) により `E` をベア名で参照できる．
- **F3（extern 宣言）**: `extern fn` の宣言は，registry エントリの `throws` 節を含めてシグネチャ一致しなければならない（MUST，0013 の一致規則の拡張）．effect ブロック（0037）内の extern も同様である：

  ```emela
  effect Http {
      extern fn request(req: Request) -> Response throws HttpError
  }
  ```

- **F4（呼び出しの意味論）**: `throws E` を持つ platform 関数の呼び出しは，0011 の throwing な呼び出しである（MUST）．したがって `?` を付けるか `try` block 内に置かなければならず，effect ブロック内の `pub fn` ラッパーが backing extern を呼ぶ場合も同様である．`@test` 本体の暗黙 try（0040 T3）は従来どおり唯一の例外である．
- **F5（ホスト契約）**: ホスト（backend / Runtime）は 1 回の呼び出しにつき，`R` の成功値または `E` の error 値のちょうど一方を返さなければならない（MUST）．想定内の失敗（接続不能・タイムアウト・不正な入力等）で trap / panic してはならない（MUST NOT）．trap は ABI 契約違反（不正なハンドル表現等）に限る．
- **F6（直交性）**: `throws` の有無は `uses` row・カバレッジ検査（0013）・capability manifest（0025）に影響しない（MUST，0009「throws と effect」の再確認）．
- **F7（進化）**: 既存エントリへの `throws` の追加・変更は破壊的変更であり，エントリの supersession として扱うべきである（SHOULD）．本仕様時点で `io.write_stdout` / `io.write_stderr` / `clock.monotonic_seconds` は非可謬のまま変更しない．
- **F8（host.\* への適用）**: embedder 定義 capability（0026）の extern も同じ規約で `throws` を宣言してよい（MAY）．

### Examples

（`fs` は未導入の仮想例である．導入仕様は別途起草される．）

registry エントリと extern 宣言：

```text
fs.read : (String) -> String throws FsError uses { Fs }
```

```emela
module fs

pub enum FsError {
    NotFound
    PermissionDenied
}

effect Fs {
    extern fn read(path: String) -> String throws FsError

    pub fn read_text(path: String) -> String throws FsError {
        read(path)?
    }
}
```

呼び出し側は 0011 の通常規則に従う：

```emela
import std.fs
import std.io

fn main() -> Unit uses { Fs, Io } {
    let text = try {
        Fs.read_text("config.txt")
    } catch {
        FsError::NotFound -> ""
        e -> panic("cannot read config")
    }
    Io.print(text)
}
```

最初の実適用は 0044 の `http.request : (Request) -> Response throws HttpError uses { Http }` である．

### Compilation Notes

この節は非規範的な実装上の補足である．

- **ABI**: `throws E` を持つ関数は IR 上 tagged union（内部 Ok / Err）へ lowering される（0011）．wasm backend は throwing な関数を `[ok flag][value|error]` の結果表現で扱っており，fallible extern はホスト実装が同じ表現で書き込むだけでよい．JS backend は `__rt` 実装が error 値を包んで throw する（0011 の内部例外表現）．
- **plugin 記述子**: 外部 backend の提供宣言（backend.json）は fallible extern の戻りを `Result<Unit, PlatformError>` 形の内部記法で表しており，これは同じ tagged union を指す記述であって表面構文ではない．
- **error 値の構築**: ホストは `E` の値をゲスト表現で構築する必要がある．payload 付き variant はゲスト側アロケーションを要するため，F2 が payload 型を registry 表現可能型に限定している（値マーシャリング ABI の詳細は 0013 の Open Question と共有）．

### 他仕様との関係

- **0013（Platform Functions）**: エントリ形式を F1 で拡張し，Open Question「失敗を返す platform 関数の導入時期」を解決する．§error との分離の「`Result` / `Option` を戻り値とする形で後続仕様において追加する」という文言は本仕様が置き換える．capability 検査・供給契約は不変．
- **0011（Error handling）**: 呼び出し側の規則（`?` / `try`–`catch`・exhaustiveness・`main` の `throws Never`）を一切変更しない．本仕様はホスト失敗を 0011 のチャネルへ接続するだけである．
- **0008（Function Types with Effects）**: 関数型の `throws E` 節の文法をそのまま extern に用いる．新文法はない．
- **0009（Effect Semantics）**: error と effect の直交（F6）．
- **0025（Capability Manifest）**: manifest の形式・内容に変更はない．
- **0026（Embedder-Defined Capabilities）**: host.\* extern への適用（F8）．
- **0038（Core Package Embedding）**: `E` の宣言位置（F2）は embedded core の境界規則に従う．
- **0040（Unit Testing）**: テスト本体では暗黙 try（T3）が fallible platform 呼び出しにもそのまま適用される．

### Open Questions

- 標準出力の失敗（EPIPE 等）を将来 `io.*` に導入するか（F7 の supersession の最初の事例になりうる）．
- 「不在」を返す platform 関数（例：環境変数の取得）に `Option<T>` 戻りの規約を並置するか．
- `E` の payload に許す型の正確な集合（値マーシャリング ABI，0013 の Open Question と共有）．
- ホストがゲストヒープへ error payload を構築する際のアロケータ契約．
