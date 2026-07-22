## 0046: HTTP Server — Socket 上の派生 effect HttpServer と main ループ

Status: Draft

HTTP サーバーを定める仕様．`std.http`（0044）に effect **`HttpServer`** を追加し，`fn main()` から回る自己末尾再帰のループ（0045）としてサーバーを書く．**`HttpServer` はプリミティブなホスト capability ではなく，`Socket`（0050）の上に書かれた派生 effect（0049）である**：その handler が `Socket.*` で接続を受け，`Bytes`（0051）を HTTP としてパース／シリアライズする．ホストが供給するのは標準 `wasi:sockets` だけであり，生成物は標準 WASI の葉のみを import する（独自 `emela_http` は生じない）．クライアント（0044 の `Http`）とは別で，`Http` は TLS がホスト責務ゆえプリミティブのまま据え置く（本改訂は 2026-07-22 の方針転換：旧版の「ホスト水準 HTTP・host platform 関数 4 本」を撤廃する）．

### Summary

- `HttpServer` を **派生 effect**（0049）として `std.http` に置く．操作 `bind` / `accept` / `respond` / `close` はシグネチャのみで，実体は**同一モジュールの handler**（`handler for HttpServer uses { Socket }`）が供給する．旧版の platform 関数 `http.server_*` 4 本は**撤廃**する．
- handler は `Socket`（0050）で listen / accept / read / write / close し，`Bytes`（0051）を HTTP/1.1 としてパース／シリアライズする．HTTP プロトコル処理は**ホストではなく Emela 側（stdlib の純関数）**が行う．
- `uses { HttpServer }` は 0049 の discharge により **leaf row `{ Socket }`** に落ちる．manifest（0025）には `socket.*` が現れ，`HttpServer` は現れない（派生 effect ゆえ）．
- サーバーは通常のプログラムである：`main` が `bind` し，自己末尾再帰のループ（0045）で `accept` → ハンドラー → `respond` を回す．リクエスト処理は逐次である（async は 0000 の非目標のまま）．
- `Server` / `Incoming` は `Socket` のハンドル（`Listener` / `Connection`）と解析済み `Request` を包む通常の record である．
- `accept` が返す `Request`，`respond` が送る `Response` の表現規則（ヘッダー小文字化・UTF-8 body 等）は 0044 と同一である．不正なリクエストは handler が処理し，ユーザーコードに届かない．

### Motivation

1. **標準 WASI で動く standalone な生成物**．旧版は `bind`/`accept`/`respond`/`close` を独自のホスト関数（`emela_http`）に lower していた．これを供給するのは `emela run`（wasmi）のホストだけで，標準 WASI ランタイム（wasmtime 等）では解決できず，`emela build` の生成物が動かなかった（0013:130「生成モジュールが import するのは WASI の関数のみ」違反）．0049（effect のコンパイル時 DI）と 0050（`Socket`）により，HTTP サーバーを「標準 `wasi:sockets` ＋ Emela の handler」に分解でき，生成物は標準 WASI の葉のみを import する．
2. **プログラムがループを所有する**．検討した対案は wasi:http `incoming-handler` 型の「ハンドラーエクスポート」——`fn (Request) -> Response` をプログラムがエクスポートし，accept ループはランタイムが所有する形——だった．これはランタイム側の並行化・per-request 隔離に向くが，`main` を持たない新しいプログラム種・専用実行モードを要し，リクエスト間で状態を持てない（stateless）．Emela は「プログラムがループを所有する，`main` から回る通常のコード」を選ぶ．`Socket.accept` は Emela 側のループが呼び，構成（ポート等）も通常の値としてコードに現れる．ハンドラーエクスポート型は本仕様と排他ではなく，別仕様として追加できる（Open Questions）．
3. **生ソケット上の HTTP（旧版からの転換）**．旧版は「最小権限・TLS・同期制約」を理由に HTTP をホスト水準 capability とし，accept が解析済み `Request` を返す形にしていた．本改訂はこれを転換し，**HTTP を `Socket` 上の Emela 実装**とする．転換の代償と対処：
   - **権限粒度**：leaf capability は `Socket`（任意 TCP）になり，manifest 上は「HTTP serve だけ」より広い．source の `uses { HttpServer }` が intent を示すが，強制される権限は `Socket` である（Open Questions：派生 effect を manifest に注記情報として残すか）．
   - **TLS**：`Socket` に TLS は無い（0050）．HTTPS は前段の TLS 終端（リバースプロキシ／ロードバランサ）に委ねる．クライアント `Http`（0044）は TLS がホスト責務ゆえプリミティブのまま据え置く．
   - **同期**：逐次処理は不変である（0000——async は非目標）．
4. **クライアントと別の権限**．「外へ接続する」（`Http`，primitive）と「listen して応答する」（`HttpServer` → `Socket`）は別の権限として現れる．serve するだけのプログラムの manifest に `Http` は現れない．プロキシは両方を宣言する（Examples）．
5. **無限ループの前提**．accept ループは自己末尾再帰で書く（0045）．`serve_loop` は 0045 の規範例そのものである．

### Specification

以下，キーワードは RFC 2119 に従う．

#### モジュールインターフェース（`std.http` への追加）

`HttpServer` は派生 effect（操作はシグネチャのみ），実体は同一モジュールの handler が `Socket` の上で供給する（0049 H1/H4）．

```emela
module http

import std.socket
import std.bytes

pub record Server {
    listener: Listener        -- 0050 の Socket ハンドル
}

pub record Incoming {
    conn: Connection          -- 0050 の Socket ハンドル
    request: Request          -- 0044 の解析済みリクエスト
}

-- 派生 effect（シグネチャのみ．extern を持たない ＝ 0049 の派生 effect）
effect HttpServer {
    pub fn bind(port: Int) -> Server throws HttpError
    pub fn accept(server: Server) -> Incoming throws HttpError
    pub fn respond(incoming: Incoming, response: Response) -> Unit throws HttpError
    pub fn close(server: Server) -> Unit throws HttpError
}

-- 同一モジュールの default handler（0049 H4）．Socket の上に HTTP を実装する
handler for HttpServer uses { Socket } {
    fn bind(port: Int) -> Server throws HttpError {
        -- Socket.listen を呼び，SocketError を HttpError にマップ
        ...
    }
    fn accept(server: Server) -> Incoming throws HttpError {
        -- Socket.accept → Socket.read で完全なリクエストを読み，
        -- Bytes を HTTP/1.1 としてパースして Request を作る（stdlib の純関数）
        ...
    }
    fn respond(incoming: Incoming, response: Response) -> Unit throws HttpError {
        -- Response を Bytes にシリアライズして Socket.write
        ...
    }
    fn close(server: Server) -> Unit throws HttpError {
        -- Socket.close(server.listener.id)
        ...
    }
}
```

HTTP のパース／シリアライズは `std.http` の純関数（`uses {}`）として供給する（例：`parse_request(raw: Bytes) -> Request throws HttpError`，`serialize_response(res: Response) -> Bytes`）．これらは handler の本体からのみ使われる実装詳細である．

#### 意味論（S 規則）

- **S1（派生 effect・leaf は Socket）**: `HttpServer` は 0049 の派生 effect である．その handler は `uses { Socket }` を持ち，`HttpServer` を使うコードの leaf row は discharge により `{ Socket }` になる（0049 D2）．`HttpServer` 操作は `Http`（0044）を要求しない（MUST）：クライアントとサーバーは別の権限として現れる．
- **S2（bind）**: `bind(port)` は handler が `Socket.listen(port)` を呼び，`Server { listener }` を返す．バインド先の既定・公開範囲は `Socket`（0050）とホスト設定に従う．`Socket` の失敗は `HttpError`（`BindFailed` 等）にマップする（MUST）．
- **S3（accept）**: `accept(server)` は handler が `Socket.accept(server.listener)` で接続を得，`Socket.read` を繰り返して完全な HTTP/1.1 リクエストを読み切り，`Bytes` をパースして `Request` を持つ `Incoming` を返す（MUST）．ユーザーコードに届く前の前処理：
  - `Method`（0044）に対応しないメソッドには handler が 501 を応答し，`accept` へ届けない（MUST）．
  - body が UTF-8 として不正なリクエストには handler が 400 を応答し，`accept` へ届けない（MUST）．
  - 解析不能・実装定義の上限（サイズ等）超過には handler が 400 / 413 等を応答してよい（MAY）．
  - 上記の自動応答の後，handler は次の接続の受付を続ける（未達リクエストはユーザーコードに現れない）．
  - `Request.url` は origin-form（パス + 任意の `?query`）である（MUST）．ヘッダー名の小文字化は 0044 H5 と同一である（MUST）．
- **S4（respond）**: `respond(incoming, response)` は handler が `Response` を `Bytes` にシリアライズし `Socket.write(incoming.conn, ...)` する．1 つの `Incoming` に対してちょうど 1 回呼ばなければならない（MUST）．2 回目以降・接続喪失は `HttpError::ConnectionClosed`（MUST）．応答の表現規則（ヘッダー・`content-length` 付与・UTF-8 body）は 0044 H5 / H6 と同一である（MUST）．応答後の接続の扱い（keep-alive・close）は handler の責務であり観測不能である．
- **S5（close）**: `close(server)` は handler が `Socket.close(server.listener.id)` で listen を終える．以後その `Server` への操作は `HttpError::ConnectionClosed` で失敗する（MUST）．
- **S6（ハンドル）**: `Server` / `Incoming` は `Socket` のハンドル（`Listener` / `Connection`）を包む通常の record である．失効・偽造ハンドルに対する安全性は `Socket`（0050 P8）に従い，対応する操作は `HttpError` で失敗する（MUST）．権限は effect row（source の `uses { HttpServer }`，leaf の `{ Socket }`）が担い，ハンドル値は権限ではない．
- **S7（逐次処理）**: リクエストは `accept` を呼ぶたびに 1 つずつ取り出され，逐次処理される．ユーザーコードの並行実行は存在しない（0000——async は非目標）．
- **S8（失敗とプログラムの寿命）**: `accept` / `respond` の error は通常の `throws HttpError` であり，`try` / `catch` で処理してループを継続できる．ユーザーコードの `panic` は 0011 どおりプログラムを終了させる．

### Examples

最小のサーバー．`handle` / `serve_loop` は旧版と同じソースで書ける（`uses { HttpServer }` は leaf `{ Socket }` に discharge される）：

```emela
import std.io
import std.http

fn handle(req: Request) -> Response uses {} {
    if req.url == "/" {
        Response { status: 200, headers: [], body: "hello from Emela\n" }
    } else {
        Response { status: 404, headers: [], body: "not found\n" }
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

このプログラムの manifest（0025．`HttpServer` は派生 effect ゆえ現れず，leaf の `socket.*` と `io.*` が現れる。外向き通信の `Http` は**現れない**）：

```json
{"format":1,"requires":["socket.accept","socket.close","socket.listen","socket.read","socket.write","io.write_stderr"],"capabilities":["Socket","Io"],"intrinsics":["…"],"entry":true}
```

上流 API を呼ぶプロキシは `Http`（primitive）も使う——manifest に `Http` と `Socket` の両方が現れる：

```emela
fn proxy(req: Request) -> Response uses { Http } {
    try {
        Http.get("https://internal.example" ++ req.url)
    } catch {
        e -> Response { status: 502, headers: [], body: "upstream failed\n" }
    }
}
```

ハンドラーは通常の関数なので，`@test`（0040）からソケットなしで直接テストできる：

```emela
import brix.assert

@test
fn root_returns_200() -> Unit uses {} {
    let res = handle(Request { method: Method::Get, url: "/", headers: [], body: "" })
    assert.eq(res.status, 200)
}
```

### Compilation Notes

この節は非規範的な実装上の補足である．

- **HTTP は Emela 側**：`HttpServer` の handler と HTTP パーサ／シリアライザ（`parse_request` / `serialize_response`）は `std.http`（embedded core，0038）の純 Emela コードである．0049 D3 により handler は IR 化前に消去され，残るのは `Socket.*` の platform 呼び出しだけになる．
- **component backend**：`Socket.*` は `wasi:sockets`（0050）へ lower する．生成 component は `wasi:sockets`（＋ `wasi:cli`）のみを import し，`emela_http` は生じない．`wasmtime run`（command world）で `main` ループが走る．
- **`emela run`（wasmi）**：`Socket.*` を当面 `std::net` でホスト実装する（旧 `http_host.rs` の HTTP 部分は Emela 側 handler へ移り，ホストは生 TCP のみ供給する）．最終的に標準 `wasi:sockets` ホストへ寄せ，Emela 固有の Rust binding を無くす（プロジェクト方針）．
- **JS / Node backend**：`Socket` を `node:net` で供給する（blocking accept への橋渡しは 0050 / 0044 と同種）．ブラウザーは `Socket` を供給しない（0013 のカバレッジ検査が reject する）．
- **ハンドラーエクスポート型との関係**：本仕様の main ループ型は wasi:http `incoming-handler` へ直接は写像しない（ループの所有者が逆である）．component/serverless 向けには将来ハンドラーエクスポート型の別仕様を追加し，同じ `Request` / `Response` 型を共有するのが自然である．

### 他仕様との関係

- **0049（Effects as Compile-Time DI）**: `HttpServer` は派生 effect であり，本仕様の handler がその実体を `Socket` の上で供給する．旧版の host platform 4 本は撤廃する．
- **0050（Socket）**: handler の依存（`uses { Socket }`）．listen / accept / read / write / close の土台．
- **0051（Bytes）**: handler が socket バイト列を HTTP としてパース／シリアライズする境界型．
- **0044（HTTP Client）**: 型（`Request` / `Response` / `Header` / `Method` / `HttpError`）と表現規則（H5 / H6）を共有する．`HttpError` のサーバー関連 variant を本仕様が使う．`Http`（client）は TLS ゆえプリミティブのまま据え置く（本仕様と対の改訂）．
- **0045（Self Tail Calls）**: accept ループの前提．`serve_loop` は 0045 の規範例である．
- **0013 / 0009**: 旧版の `http.server_*` registry エントリを撤廃する．`HttpServer` は組み込み capability 集合に**現れない**（派生 effect）．leaf の `Socket`（0050）が registry・capability 集合に現れる．
- **0038（Core Package Embedding）**: `std.http` は embedded core（0044 H9）であり，本仕様は HTTP パーサ等の純 Emela 実装を同モジュールに加える．
- **0025（Capability Manifest）**: manifest には leaf（`socket.*` / `Socket`）が現れ，`HttpServer` は現れない（0049 D5）．
- **0037（Module-Unit Imports and First-Class Effects）**: 1 モジュール複数 effect（`Http` と `HttpServer`）は 0037 の通常規則である．
- **0011 / 0040**: `throws` / `try`–`catch` / `panic`・ハンドラーの直接テストは不変．

### Open Questions

- **権限粒度**：leaf が `Socket`（任意 TCP）になることで manifest の精密さが「HTTP serve だけ」より広がる．派生 effect（`HttpServer`）を manifest に**注記情報**として残し，「この Socket 利用は HTTP サーバーである」を機械可読にするか．
- **TLS 終端**：前段プロキシ前提でよいか，将来 `Tls` primitive effect（wasi:tls 系）を導入して `HttpServer` を `Tls`/`Socket` 上に載せられるようにするか．
- **body の `Bytes` 化**：現状 `Request`/`Response` の body は UTF-8 `String`（0044）．バイナリ body のため body を `Bytes`（0051）にする 0044 改訂を行うか．
- **keep-alive・接続再利用**：handler の respond 後の接続処理（現状は実装定義）を規定するか．
- **サーバー構成**：バインド先・同時接続数・タイムアウトを `bind` 引数／設定 record にするか，ホスト（`Socket`）ポリシーに留めるか．
- **ハンドラーエクスポート型・並行処理**：component/serverless 向けの別仕様（Motivation 2 の対案）を追加するか，将来の async capability（0000/0049 Open Q3）を待つか．
- **インメモリテストサーバー**：`HttpServer` handler をスコープ限定で差し替える（0049 Open Q1）ことで，Effect-TS の `layerTest` 相当のモックサーバーを与えるか．
