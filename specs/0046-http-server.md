## 0046: HTTP Server — effect HttpServer と main ループ

Status: Draft

サーバー専用の構文・実行モードは導入せず，`fn main()` のループで回す（2026-07-19 決定）．

HTTP サーバーを定める仕様．`std.http`（0044）に 2 つ目の effect **`HttpServer`** を追加し，`bind` / `accept` / `respond` / `close` の platform 関数（0013 / 0043）で，通常の Emela コード——`fn main()` から呼ばれる自己末尾再帰のループ（0045）——としてサーバーを書く．専用の属性・エントリポイント・CLI モードは導入しない．クライアント（0044 の `Http`）とは別 capability であり，serve するだけのプログラムの manifest（0025）に外向き通信の権限は現れない．

### Summary

- 組み込み capability に `HttpServer` を追加する．`std.http`（0044）が同じモジュール内に effect `HttpServer` と record `Server` / `Incoming` を公開する（0037：1 モジュール複数 effect）．
- platform 関数は 4 本：`http.server_bind` / `http.server_accept` / `http.server_respond` / `http.server_close`．すべて fallible（0043）で，error 型はクライアントと共有の `HttpError`（0044）である．
- サーバーは通常のプログラムである：`main` が `bind` し，自己末尾再帰のループ（0045）で `accept` → ユーザーコード → `respond` を回す．リクエストの処理は逐次である（async は 0000 の非目標のまま）．
- `accept` が返す `Request`，`respond` が送る `Response` の表現規則（ヘッダー小文字化・UTF-8 body 等）は 0044 と同一である．不正なリクエストはホストが処理し，ユーザーコードに届かない．
- `Server` / `Incoming` はホスト発行の識別子を包む通常の record である．不透明型は導入しない．偽造・失効した識別子はホストが実行時に `HttpError` で拒否する．

### Motivation

1. **サーバーは通常のコードとして書く**．検討した対案は wasi:http `incoming-handler` 型の「ハンドラーエクスポート」——指定した `fn (Request) -> Response` をプログラムがエクスポートし，accept ループはランタイムが所有する形——だった．ホスト側での並行化・リクエスト単位の隔離・serverless 環境への適合という利点はあるが，専用属性・専用実行モード・「`main` の任意化」という新しい言語表面を要する．Emela は「プログラムがループを所有する，`main` から回る通常のコード」を選ぶ（2026-07-19 決定）：新しいプログラム種を作らず，ポートなどの構成も通常の値としてコードに現れ，実行は `emela run` のままである．ハンドラーエクスポート型は本仕様と排他ではなく，ホスト並行化が必要になった時点で別仕様として追加できる（Open Questions）．
2. **クライアントと別 capability**．「外へ接続する」（`Http`）と「listen して応答する」（`HttpServer`）は監査上まったく別の権限である．分離により，serve するだけのプログラムの manifest は `HttpServer` のみを示し，「このサービスは外向きに通信しない」が配布物から機械的に読める（0000 の権限表テーゼ・0025）．プロキシのように両方を使うプログラムは両方を宣言する．
3. **生ソケットではなくホスト水準の HTTP**．理由は 0044 Motivation と同じ（最小権限・TLS・同期制約）．accept が返すのはバイト列ではなく解析済みの `Request` であり，HTTP パーサ・接続管理・TLS 終端はホストの責務である．`net` は予約のまま使用しない．
4. **無限ループの前提**．Emela にはループ構文がないため，accept ループは自己末尾再帰で書く．0045 の保証（自己末尾呼び出しはスタックを消費しない）が本仕様の前提であり，`serve_loop` は 0045 の規範例そのものである．

### Specification

以下，キーワードは RFC 2119 に従う．

#### モジュールインターフェース（`std.http` への追加）

```emela
module http

pub record Server {
    id: Int
}

pub record Incoming {
    id: Int
    request: Request
}

effect HttpServer {
    extern fn server_bind(port: Int) -> Server throws HttpError
    extern fn server_accept(server: Server) -> Incoming throws HttpError
    extern fn server_respond(incoming: Incoming, response: Response) -> Unit throws HttpError
    extern fn server_close(server: Server) -> Unit throws HttpError

    pub fn bind(port: Int) -> Server throws HttpError {
        server_bind(port)?
    }

    pub fn accept(server: Server) -> Incoming throws HttpError {
        server_accept(server)?
    }

    pub fn respond(incoming: Incoming, response: Response) -> Unit throws HttpError {
        server_respond(incoming, response)?
    }

    pub fn close(server: Server) -> Unit throws HttpError {
        server_close(server)?
    }
}
```

#### registry（0013 / 0043）

```text
http.server_bind    : (Int) -> Server throws HttpError uses { HttpServer }
http.server_accept  : (Server) -> Incoming throws HttpError uses { HttpServer }
http.server_respond : (Incoming, Response) -> Unit throws HttpError uses { HttpServer }
http.server_close   : (Server) -> Unit throws HttpError uses { HttpServer }
```

#### 意味論（S 規則）

- **S1（capability）**: `HttpServer` を組み込み capability の集合（0009）に追加する．サーバー操作は capability `Http`（0044）を要求しない（MUST）：クライアントとサーバーは独立に監査される．
- **S2（bind）**: `bind(port)` は当該ポートで listen を開始し，`Server` を返す．バインド先インターフェースは loopback を既定とすべきである（SHOULD；公開バインドの構成は Open Questions）．失敗（ポート使用中・権限不足等）は `HttpError::BindFailed` である（MUST）．同時に複数の `Server` を保持してよい（MAY）．
- **S3（accept）**: `accept(server)` は，完全なリクエストを 1 つ受信するまで blocking し，解析済みの `Request` を持つ `Incoming` を返す（MUST）．ユーザーコードに届く前のホスト前処理として：
  - `Method`（0044）に対応しないメソッドのリクエストには，ホストが 501 を応答し，`accept` へは届けない（MUST）．
  - body が UTF-8 として不正なリクエストには，ホストが 400 を応答し，`accept` へは届けない（MUST）．
  - 解析不能なリクエスト・実装定義の上限（サイズ等）を超えるリクエストには，ホストが 400 / 413 等を応答してよい（MAY）．
  - `Request.url` は origin-form のリクエストターゲット（パス + 任意の `?query`）である（MUST）．ヘッダー名の小文字化は 0044 H5 と同一である（MUST）．
- **S4（respond）**: `respond(incoming, response)` は応答を送信する．1 つの `Incoming` に対してちょうど 1 回呼ばなければならない（MUST）．2 回目以降・接続が既に失われた場合は `HttpError::ConnectionClosed` である（MUST）．応答の表現規則（ヘッダー・`content-length` 等の付与・UTF-8 body）は 0044 H5 / H6 と同一である（MUST）．応答後の接続の扱い（keep-alive・close）はホストの責務であり観測不能である．
- **S5（close）**: `close(server)` は listen を終了し，以後その `Server` に対する操作は `HttpError::ConnectionClosed` で失敗する（MUST）．`respond` されていない `Incoming` の扱いは実装定義である．
- **S6（ハンドル）**: `Server` / `Incoming` はホストが発行する識別子を包む通常の record である．識別子の偽造・失効値に対して，ホストは未定義動作を起こしてはならず（MUST NOT），対応する操作を `HttpError` で失敗させなければならない（MUST）．権限は effect row（`uses { HttpServer }`）が担い，ハンドル値は権限ではない．
- **S7（逐次処理）**: リクエストはプログラムが `accept` を呼ぶたびに 1 つずつ取り出され，逐次処理される．ホストは `accept` 待ちの間の接続受付・バッファリングを行ってよい（MAY）が，ユーザーコードの並行実行は存在しない（0000——async は非目標）．
- **S8（失敗とプログラムの寿命）**: `accept` / `respond` の error は通常の `throws` であり，`try` / `catch` で処理してループを継続できる．ユーザーコードの `panic` は 0011 どおりプログラムを終了させる：リクエスト単位の隔離は存在せず，監督・再起動はホスト運用の領分である．

### Examples

最小のサーバー．`serve_loop` は 0045 の保証により一定スタックで無限に回る：

```emela
import std.io
import std.http

fn handle(req: Request) -> Response uses {} {
    if req.url == "/" {
        Response {
            status: 200
            headers: []
            body: "hello from Emela\n"
        }
    } else {
        Response {
            status: 404
            headers: []
            body: "not found\n"
        }
    }
}

fn serve_loop(server: Server) -> Unit uses { HttpServer } {
    try {
        let incoming = HttpServer.accept(server)
        HttpServer.respond(incoming, handle(incoming.request))
    } catch {
        e -> ()
    }
    serve_loop(server)  -- 自己末尾呼び出し（0045）：スタックを消費しない
}

fn main() -> Unit uses { Io, HttpServer } {
    try {
        serve_loop(HttpServer.bind(8080))
    } catch {
        e -> Io.eprint("failed to start server\n")
    }
}
```

このプログラムの manifest（0025．外向き通信の capability `Http` が**現れない**ことに注意．`intrinsics` はプログラムの到達集合に依存するため省略記法で示す）：

```json
{"format":1,"requires":["http.server_accept","http.server_bind","http.server_respond","io.write_stderr"],"capabilities":["HttpServer","Io"],"intrinsics":["…"],"entry":true}
```

上流 API を呼ぶプロキシは両方の capability を宣言する——manifest に `Http` と `HttpServer` の両方が現れ，「外向きに通信するサービス」であることが配布物から読める：

```emela
fn proxy(req: Request) -> Response uses { Http } {
    try {
        Http.get("https://internal.example" ++ req.url)
    } catch {
        e -> Response {
            status: 502
            headers: []
            body: "upstream failed\n"
        }
    }
}
```

ハンドラーは通常の関数なので，`@test`（0040）からソケットなしで直接テストできる（同居テスト——`handle` と同じファイルに置く）：

```emela
import brix.assert

@test
fn root_returns_200() -> Unit uses {} {
    let res = handle(Request {
        method: Method::Get
        url: "/"
        headers: []
        body: ""
    })
    assert.eq(res.status, 200)
}
```

### Compilation Notes

この節は非規範的な実装上の補足である．

- **`emela run`（wasmi）**: ホスト関数 4 本を Rust の blocking なリスナー（`std::net` + 小さな HTTP パーサ，または軽量クレート）で実装する．`server_accept` はホスト側で blocking し，解析済み `Request` をゲストヒープへ構築して返す（ホスト→ゲストのアロケーション契約は 0044 Compilation Notes と共有）．
- **JS / Node backend**: `node:http` はコールバック駆動であり，「blocking な accept」への橋渡し（`worker_threads` + `Atomics.wait` 等）が要る．クライアント（0044）と同種の工夫であり，ブラウザーターゲットは `HttpServer` を供給しない（0013 のカバレッジ検査が reject する）．
- **WAMR**: ネイティブのホスト関数として実装する．0025 の manifest（`HttpServer` の有無）をインスタンス化前の監査に使える．
- **ハンドラーエクスポート型との関係**: 本仕様の main ループ型は wasi:http `incoming-handler` へ直接は写像しない（ループの所有者が逆である）．コンポーネントモデル環境・serverless 環境向けには，将来ハンドラーエクスポート型の別仕様を追加し，同じ `Request` / `Response` 型を共有するのが自然である．

### 他仕様との関係

- **0044（HTTP Client）**: 型（`Request` / `Response` / `Header` / `Method` / `HttpError`）と表現規則（H5 / H6）を共有する．`HttpError` のサーバー専用 variant（`BindFailed` / `ConnectionClosed`）を本仕様が使う．capability は分離される（S1）．
- **0045（Self Tail Calls）**: accept ループの前提（Motivation 4）．`serve_loop` は 0045 の保証の最初の利用者である．
- **0043（Fallible Platform Functions）**: 4 本の registry エントリはすべて F 規則に従う fallible extern である．
- **0013 / 0009**: registry へのエントリ追加と組み込み capability 集合への `HttpServer` 追加（部分更新）．供給契約・カバレッジ検査は不変．
- **0038（Core Package Embedding）**: `std.http` は 0044 H9 で embedded core に加わっており，本仕様は同モジュールへアイテムを追加するだけである．
- **0025（Capability Manifest）**: `HttpServer` と `http.server_*` が manifest に現れる．形式は不変．クライアントとの capability 分離が manifest の監査価値を高める（Motivation 2）．
- **0011（Error handling）**: `throws` / `try`–`catch` / `panic` の規則は不変（S8）．`main` の `throws Never` 規則も不変であり，`bind` の失敗は `main` 内で catch する（Examples）．
- **0040（Unit Testing）**: ハンドラー関数の直接テスト（Examples）．ランナーが供給する platform は `run` と同一（T4）なので，`HttpServer` を要求するテストも書ける．
- **0037（Module-Unit Imports and First-Class Effects）**: 1 モジュール複数 effect（`Http` と `HttpServer`）は 0037 の通常規則である．
- **0000（Language Design Principles）**: 逐次処理（S7）は async 非目標の帰結である．capability 分離（S1）は「effect row = 権限表」テーゼの適用である．

### Open Questions

- バインド先インターフェース・TLS 終端・同時接続数などのサーバー構成の面（`bind` の引数を増やすか，設定 record を導入するか，ホストポリシーに留めるか）．
- `accept` のタイムアウト（現状は無期限 blocking．graceful shutdown の手段と連動する）．
- `Http.serve(port, handler)` 形の便宜ラッパー（関数値引数の effect row を多相にする必要があり，0022 の関数値との統合を待って再訪する）．
- ルーティング・ミドルウェアのヘルパー層（純 Emela の外部 Pome——brix と同様の位置づけ——が自然か）．
- リクエスト単位の隔離と並行処理．ホストが並行化できるハンドラーエクスポート型（Motivation 1 の対案）を別仕様として追加するか，将来の async capability（0000）を待つか．
- streaming（chunked・SSE）・WebSocket への拡張．
- `Server` / `Incoming` の `id` フィールドを非公開にする record フィールド可視性（0006 は未定義）の導入．
- effect handler が将来導入される場合（0000 の改訂を伴う別議論），`HttpServer` 操作の再解釈によるインメモリテストサーバー（Effect-TS の `layerTest` に相当）をどう位置づけるか．
