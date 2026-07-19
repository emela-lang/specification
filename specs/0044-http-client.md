## 0044: HTTP Client — effect Http と std.http

Status: Draft

HTTP はホスト水準の capability とし，`Io` とも生ソケットとも階層化しない（2026-07-19 決定）．

HTTP クライアントを定める仕様．組み込み capability **`Http`** と embedded core モジュール **`std.http`**（0038）を追加し，platform 関数 `http.request`（0013 / 0043）で 1 回の HTTP 交換を行う．`Http` は `Io` / `Clock` と並列の独立した capability であり，effect handler による下位 effect への解釈（0000 が不採用）でも，ゲスト側ソケットの上のライブラリでもない．サーバー側は 0046 が定める．

### Summary

- 組み込み capability に `Http` を追加する（0009 の組み込み集合の拡張）．0009 / 0013 が予約する `net`（生ソケット）は未定義のまま温存し，本仕様は使用しない．
- `std.http` を 0038 の embedded core 集合に追加する．`http` は `std` の予約モジュール名になる．
- platform 関数は 1 本である：`http.request : (Request) -> Response throws HttpError uses { Http }`．0043（fallible platform functions）の最初の適用例である．
- 呼び出しは同期であり，1 呼び出しがちょうど 1 回の HTTP request / response 交換を行う．async は導入しない（0000 の非目標のまま）．
- `Request` / `Response` / `Header` / `Method` / `HttpError` は `std.http` の通常の record / enum である（0006 / 0005）．body は `String`（UTF-8 テキスト，0007），headers は `Array<Header>` である．
- transport の失敗のみが `HttpError` になる．HTTP ステータス（非 2xx を含む）は成功した `Response` である．リダイレクトは自動追従しない．
- TLS（`https`）はホストが提供・検証する．言語・ゲスト側に証明書の面は存在しない．

### Motivation

1. **対外通信の需要**．Emela プログラムが外部 API を呼び，サーバー（0046）を書けるようにする．HTTP は失敗が本質的な最初の標準 capability であり，0043 の規約を最初に使う．
2. **`Io` の上でも handler でもない**．Emela の effect は「要求の追跡と境界での供給」であり（0000 / 0013），effect row は型検査後に消去される．Koka / Unison のような handler による再解釈（`Http` を `Io` へ解釈する層）は機構として存在しない．`uses` row = ホスト import 集合 = サンドボックスの権限表（0000）という中心テーゼに従えば，`Http` は独自の platform 関数を持つ独立 capability として供給されるときにのみ意味を持つ：manifest（0025）の `Http` が「このモジュールは HTTP 通信をする」をインスタンス化前に監査可能にする．
3. **生ソケットではなくホスト水準の HTTP**．HTTP を「`net`（TCP）上の Emela ライブラリ」にしない理由：
   - **最小権限**．`uses { Http }` は「任意のソケット通信」より狭く，監査者にとって意味のある宣言である．HTTP-over-`net` は精密な capability を不透明な汎用権限へ洗い流してしまう．
   - **TLS はホストの責務**．証明書ストア・検証・失効はホスト環境の資産であり，ゲスト内 TLS スタックは WAMR の小フットプリント方針（0000）と両立しない．
   - **同期制約との整合**．「1 回の blocking なホスト呼び出し = 1 交換」は，async を持たない現行実行モデル（0000）でそのまま実装できる粒度である．
   - 同じ判断の前例が wasi:http である（HTTP をソケットライブラリではなくホストインターフェースとして定義する）．
4. **他言語の前例**．Koka は `io` を `net` / `fsys` 等の細粒度 effect へ分解しており，capability を protocol 粒度で分ける方向の先例である．Unison は `Http` ability を handler で `IO` へ解釈する——機構は 0000 と非互換だが，「HTTP を独立した能力として宣言する」API 形状は本仕様と一致する．Effect-TS は `HttpClient` を注入されるサービスとし，インターフェースと実装（platform 層）を分離する——Emela では registry / backend 供給（0013）が同じ役割を果たす．OCaml eio は「必要な capability だけを渡す」ことで監査性を得る——`uses { Http }` はその静的版である．

### Specification

以下，キーワードは RFC 2119 に従う．

#### モジュールインターフェース

`std.http` は少なくとも次を公開する（規範）．クライアント部分のみを示す（サーバー部分は 0046）：

```emela
module http

pub enum Method {
    Get
    Head
    Post
    Put
    Delete
    Patch
    Options
}

pub record Header {
    name: String
    value: String
}

pub record Request {
    method: Method
    url: String
    headers: Array<Header>
    body: String
}

pub record Response {
    status: Int
    headers: Array<Header>
    body: String
}

pub enum HttpError {
    InvalidUrl(String)
    ConnectFailed(String)
    Timeout
    TooLarge
    NonUtf8Body
    Protocol(String)
    BindFailed(String)
    ConnectionClosed
}

effect Http {
    extern fn request(req: Request) -> Response throws HttpError

    pub fn send(req: Request) -> Response throws HttpError {
        request(req)?
    }

    pub fn get(url: String) -> Response throws HttpError {
        request(Request {
            method: Method::Get
            url: url
            headers: []
            body: ""
        })?
    }

    pub fn post(url: String, body: String) -> Response throws HttpError {
        request(Request {
            method: Method::Post
            url: url
            headers: []
            body: body
        })?
    }
}
```

- 公開素通し操作の名前が `send` なのは，backing extern `request` と同一ブロック内で名前が衝突するためである（0037）．
- `BindFailed` / `ConnectionClosed` はサーバー側（0046）の操作が送出する variant であり，`http.request` は送出しない（MUST NOT）．
- importer は 0037 R2(b) により型（`Request` / `Response` / `Header` / `Method` / `HttpError`）をベア名で，操作を `Http.get(...)` の形で参照する．

#### registry（0013 / 0043）

```text
http.request : (Request) -> Response throws HttpError uses { Http }
```

#### 意味論（H 規則）

- **H1（capability）**: `Http` を組み込み capability の集合（0009）に追加する．manifest（0025）の capability 識別子は `"Http"`，platform 関数の修飾名は `http.request` である（0037：canonical 名はモジュール名修飾）．
- **H2（同期・1 交換）**: `http.request` はちょうど 1 回の HTTP request / response 交換を行い，応答の完了または失敗まで blocking する（MUST）．platform による接続の再利用（keep-alive・プール）は観測不能である限り行ってよい（MAY）．
- **H3（リダイレクト）**: platform はリダイレクトを自動追従してはならない（MUST NOT）．3xx は通常の `Response` として返る．追従はユーザーコード（再帰）で書ける．
- **H4（ステータスは error ではない）**: platform は，HTTP 応答が得られた場合，そのステータスによらず（非 2xx を含め）`Response` を返さなければならない（MUST）．`HttpError` は transport 水準の失敗（接続・時間・プロトコル違反・表現不能な応答）に限る（MUST）．
- **H5（ヘッダー）**: ヘッダー名は要求・応答の両方向で小文字に正規化される（MUST）．順序と同名の重複は保存される．platform はプロトコル上必須のヘッダー（`host`・`content-length` 等）を `url` / `body` から導出して付与し，これらに対するユーザー指定値を無視すべきである（SHOULD）．
- **H6（body はテキスト）**: 要求 body は `body` の UTF-8 バイト列として送信される（MUST）．応答 body が UTF-8 として不正な場合，呼び出しは `HttpError::NonUtf8Body` で失敗しなければならない（MUST）．（バイナリ body は将来の `Bytes` 型の導入を待つ——Open Questions．）
- **H7（タイムアウト）**: platform は実装定義のタイムアウトを課すべきであり（SHOULD），超過は `HttpError::Timeout` として報告しなければならない（MUST）．
- **H8（TLS）**: `https` URL は platform が TLS を提供する場合に限りサポートされる．証明書検証はホストポリシーであり，既定で有効でなければならない（MUST）．検証を無効化する面は言語・`std.http` に存在しない（MUST NOT）．
- **H9（embedding と予約名）**: `std.http` は 0038 の embedded core 集合に加わる（0038 の「将来のモジュール追加（`fs`, `random` 等）は本仕様を supersede せず拡張する．追加の時期・内容は各機能仕様が定める」に基づく拡張である）．`http` は `std` の予約モジュール名となり，0038 の予約名規則（サイレントシャドウ禁止）が適用される（MUST）．
- **H10（URL）**: `url` は絶対 `http` / `https` URL でなければならない（MUST）．それ以外は `HttpError::InvalidUrl` で失敗する（MUST）．
- **H11（error variant の意味）**:

  | variant | 意味 |
  |---|---|
  | `InvalidUrl(String)` | `url` が H10 を満たさない（payload は当該 url） |
  | `ConnectFailed(String)` | 名前解決・TCP 接続・TLS ハンドシェイクの失敗 |
  | `Timeout` | H7 のタイムアウト超過 |
  | `TooLarge` | 実装定義のサイズ上限超過 |
  | `NonUtf8Body` | 応答 body が UTF-8 でない（H6） |
  | `Protocol(String)` | HTTP プロトコル違反・解釈不能な応答 |
  | `BindFailed(String)` | サーバー専用（0046）．`http.request` は送出しない |
  | `ConnectionClosed` | サーバー専用（0046）．`http.request` は送出しない |

### Examples

GET して本文を表示する（0011 の通常規則——throwing な呼び出しは `try` 内に置く）：

```emela
import std.io
import std.http

fn main() -> Unit uses { Io, Http } {
    let body = try {
        Http.get("https://example.com/").body
    } catch {
        HttpError::Timeout -> "request timed out\n"
        e -> "request failed\n"
    }
    Io.print(body)
}
```

ヘッダーを明示する要求は `Http.send` で組み立てる：

```emela
fn fetch_json(url: String) -> Response throws HttpError uses { Http } {
    Http.send(Request {
        method: Method::Get
        url: url
        headers: [Header {
            name: "accept"
            value: "application/json"
        }]
        body: ""
    })?
}
```

非 2xx は成功した `Response` である（H4）：

```emela
fn exists(url: String) -> Bool throws HttpError uses { Http } {
    let res = Http.get(url)?
    res.status != 404
}
```

上の `main` の manifest（0025．`intrinsics` はプログラムの到達集合に依存するため省略記法で示す）：

```json
{"format":1,"requires":["http.request","io.write_stdout"],"capabilities":["Http","Io"],"intrinsics":["…"],"entry":true}
```

### Compilation Notes

この節は非規範的な実装上の補足である．

- **構造化された境界**: `http.request` は record / enum / `Array` が platform 境界を渡る最初のエントリである．引数はゲストの既存メモリ表現をホストが読み，応答はホストがゲストヒープへ構築する（ホスト→ゲストのアロケーション契約が新規に要る）．値マーシャリング ABI は 0013 の Open Question と共有し，本仕様では固定しない．
- **`emela run`（wasmi）**: ホスト関数として blocking な Rust HTTP クライアントで実装するのが自然である．同期モデルとの相性は最も良い．
- **JS / Node backend**: 同期呼び出しの実装には工夫が要る（`worker_threads` + `Atomics.wait` で `fetch` を橋渡しする等）．ブラウザーターゲットは `Http` を供給しない選択ができ，その場合 `Http` を要求する executable は 0013 のカバレッジ検査がコンパイル時に reject する——これは設計どおりの挙動である．
- **WAMR**: ネイティブのホスト関数（libcurl・プラットフォーム HTTP スタック等）で供給し，0025 の manifest をインスタンス化前の監査に使う．

### 他仕様との関係

- **0043（Fallible Platform Functions）**: `http.request` は `throws` を持つ最初の registry エントリである（F1–F5）．
- **0013（Platform Functions）**: registry へのエントリ追加（供給契約・カバレッジ検査は不変）．
- **0009（Effect Semantics）**: 組み込み capability 集合へ `Http` を追加する（部分更新）．`net` は予約のまま不変．
- **0038（Core Package Embedding）**: embedded core 集合へ `std.http` を追加し，予約名に `http` を加える（H9）．0038 の Open Question「`net` 等の導入仕様との接続」の HTTP 部分に答える．
- **0037（Module-Unit Imports and First-Class Effects）**: effect 宣言・`Http.op(...)` 修飾・`uses` ゲート・型のベア名 import はすべて 0037 の通常規則である．本仕様は名前解決規則を追加しない．
- **0011（Error handling）** / **0008**: `throws HttpError` は通常の error channel である．
- **0006 / 0028 / 0005**: `Request` 等は通常の record / enum である．標準ライブラリで record を使う最初の仕様になる（0006 は Draft——構築とフィールドアクセスのみに依存する）．
- **0007（Arrays and Strings）**: headers は `Array<Header>`（embedded core は外部パッケージの `List`（0029）に依存できないため），body は UTF-8 テキストの `String`．
- **0025（Capability Manifest）**: capability `"Http"` / 修飾名 `http.request` が manifest に現れる．形式は不変．
- **0026（Embedder-Defined Capabilities）**: `Http` は標準 capability であり `host.*` ではない．テストでの差し替え（mock）は 0026 の capability 差し替えで可能である．
- **0032（Packaging）**: 0032 の例が想像した「`net, clock` を要求するサードパーティ http パッケージ」は，標準の監査可能な capability としての `Http` に置き換わる．
- **0046（HTTP Server）**: 同じ `std.http` にサーバー側の effect `HttpServer` を追加する．`HttpError` のサーバー専用 variant は 0046 が使う．

### Open Questions

- streaming（chunked な送受信）と `Bytes` 型の導入．導入後も `http.request` の「全量 `String`」意味論は保つか，新エントリを足すか．
- タイムアウト・プロキシ等の設定面（`Request` のフィールドにするか，ホストポリシーに留めるか）．
- `Method` に文字列の escape hatch（非標準メソッド）を許すか．
- cookie・認証ヘルパー・URL パーサーを `std.http` に置くか，純 Emela の外部 Pome に置くか．
- 同名ヘッダーの多値の扱い（結合か列挙か——現状は列挙）．
- 要求・応答サイズの上限の既定値（`TooLarge` の閾値）．
- 将来 `Net`（生ソケット）仕様が導入されても，`http.request` をゲスト側ソケット実装へ置き換えない（TLS・監査の理由が消えない）ことの明文化．
- JSON エンコード / デコードの提供層．
- effect handler が将来導入される場合（0000 の改訂を伴う別議論），`Http` 操作の handler による再解釈（テストのモック等）をどう位置づけるか．現行のモックは capability 差し替え（0026）で行う．
