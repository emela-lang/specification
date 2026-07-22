## 0050: Socket — a Primitive Effect over wasi:sockets

Status: Draft

生 TCP を扱うプリミティブ effect **`Socket`** を定義する仕様。ホストが `wasi:sockets`（WASI 0.2）で供給する最小のバイト境界（listen / accept / read / write / close）を、0013 の platform 関数として公開する。`Socket` は 0049 の意味で **プリミティブ effect（extern バッキング）** であり、`HttpServer`（0046 改訂）はこの上に **派生 effect の handler** として書かれる。0009 が予約したまま未定義だった `net`（生ソケット）の TCP 部分を実体化する。TLS は含めない（ホスト／前段プロキシの責務）。

### Summary

- 組み込み capability に **`Socket`** を追加する（0009 の組み込み集合の拡張）。`Socket` は `Io` / `Clock` と並列の独立した primitive capability である。
- platform 関数は最小の TCP 境界：`socket.listen` / `socket.accept` / `socket.read` / `socket.write` / `socket.close`。すべて fallible（0043）で、error 型は `SocketError`。
- `Listener` / `Connection` はホストが発行する識別子を包む通常の record である（0046 の `Server` / `Incoming` と同じ扱い）。偽造・失効した識別子はホストが実行時に `SocketError` で拒否する（不透明型は導入しない）。
- 読み書きするバイト列の型は **`Bytes`** とする（本仕様の依存。0044 Open Question の `Bytes` 導入に接続する。暫定表現は Open Questions）。
- **TLS は含めない**。`Socket` は平文 TCP のみを供給する。HTTPS を要する用途は前段の TLS 終端（リバースプロキシ／ロードバランサ）に委ねる。クライアント HTTP（`Http`、0044）は TLS がホスト責務であるため `Socket` の上に置かず primitive のまま（0044:224-225 と整合）。
- lower 先は component backend の `wasi:sockets`（WASI 0.2）。`emela run`（wasmi）は当面ホスト実装（`std::net`）で供給してよい。

### Motivation

0046（HTTP Server）は当初、`bind` / `accept` / `respond` を**ホスト水準の HTTP** の platform 関数として供給していた。これは標準 WASI に無い独自 import（`emela_http`）を生み、生成 `.wasm` が標準 WASI ランタイムで動かない原因になっていた（0013:130 違反）。

0049 が「プリミティブ effect の上に派生 effect の handler を静的に載せる」機構を与えたことで、HTTP サーバーを「最小の primitive（`Socket`）＋ Emela の handler（HTTP パース）」に分解できる。ホストが供給するのは標準 `wasi:sockets` だけになり、生成物は標準 WASI の葉のみを import する。accept ループと HTTP プロトコル処理は Emela 側で行い、監査可能な権限は `Socket`（TCP）に落ちる。

`Socket` は 0009 が「予約のまま未定義」としてきた `net` の、TCP に限った最小実体化である。UDP・名前解決・より広いソケット操作は本仕様では扱わない（Open Questions）。

### Specification

以下、キーワードは RFC 2119 に従う。

#### effect インターフェース（`std.socket`）

```
module socket

record Listener { id: Int }
record Connection { id: Int }

enum SocketError {
    BindFailed(String)
    AcceptFailed(String)
    ConnectionClosed
    Io(String)
}

effect Socket {
    extern fn listen(port: Int) -> Listener throws SocketError
    extern fn accept(listener: Listener) -> Connection throws SocketError
    extern fn read(conn: Connection, max: Int) -> Bytes throws SocketError
    extern fn write(conn: Connection, data: Bytes) -> Unit throws SocketError
    extern fn close(handle: Int) -> Unit
}
```

- **P1（capability）**: `Socket` を組み込み capability 集合（0009）に追加する。`Socket` 操作は他の capability（`Io` 等）を要求しない（MUST）。`Socket` は 0049 の意味でプリミティブ effect（`extern fn` バッキング）である。
- **P2（platform 関数）**: 各操作は 0013 の platform 関数に lower する。canonical 名は `socket.listen` / `socket.accept` / `socket.read` / `socket.write` / `socket.close`。すべて fallible（0043）で error 型は `SocketError`（`close` は infallible とする）。
- **P3（listen）**: `listen(port)` は指定ポートで TCP を bind + listen し、`Listener` を返す。bind 失敗は `SocketError::BindFailed`（MUST）。実装定義の backlog を用いてよい（MAY）。
- **P4（accept）**: `accept(listener)` は次の受付済み接続を1つ取り出し `Connection` を返す。接続が来るまでブロックする（同期。async は 0000 の非目標のまま）。listener が閉じられていれば `SocketError::ConnectionClosed`（MUST）。
- **P5（read）**: `read(conn, max)` は最大 `max` バイトを読み、`Bytes` を返す。相手が正常に閉じた場合は長さ 0 の `Bytes` を返す（EOF、MUST）。それ以外の失敗は `SocketError::Io` / `ConnectionClosed`。
- **P6（write）**: `write(conn, data)` は `data` を書き切る（部分書き込みはホストが吸収し、全量書くか error にする、MUST）。接続喪失は `SocketError::ConnectionClosed`。
- **P7（close）**: `close(handle)` は `Listener.id` または `Connection.id` を受け、その資源を解放する。以後その識別子への操作は `SocketError::ConnectionClosed` で失敗する（MUST）。二重 close は無害（MUST）。
- **P8（ハンドル）**: `Listener` / `Connection` はホスト発行の識別子（`id`）を包む通常の record である。識別子の偽造・失効値に対して、ホストは未定義動作を起こしてはならず（MUST NOT）、対応する操作を `SocketError` で失敗させなければならない（MUST）。権限は effect row（`uses { Socket }`）が担い、ハンドル値は権限ではない（0046 S6 と同旨）。

#### バイト列（`Bytes`）

- **B1**: `read` / `write` が扱う型を `Bytes`（不変のバイト列）とする。`Bytes` の定義（構築・添字・連結・`String` との相互変換・表現）は **0051** が与える。

### Examples

エコーサーバーの1接続分（`main` から回すループは 0045／0046 の形。HTTP パースは含まない生 TCP の例）：

```
import std.socket

fn serve(listener: Listener) -> Unit uses { Socket } {
    try {
        let conn = Socket.accept(listener)
        let data = Socket.read(conn, 4096)
        Socket.write(conn, data)          -- エコー
        Socket.close(conn.id)
    } catch { e -> () }
    serve(listener)                        -- 自己末尾呼び出し（0045）
}

fn main() -> Unit uses { Socket } {
    try {
        serve(Socket.listen(8080))
    } catch { e -> () }
}
```

この `main` の leaf row は `{ Socket }`（プリミティブ）であり、manifest は `socket.*` を列挙する。生成 component は `wasi:sockets` のみを import する（`emela_http` は現れない）。

### Compilation Notes

この節は非規範的な実装上の補足である。

- **component backend（`wasi:sockets`）**: `socket.*` は WASI 0.2 の `wasi:sockets/tcp`・`wasi:io/streams` へ lower する。`listen` は `tcp-create-socket` + `bind` + `listen`、`accept` は `tcp-socket.accept`（`input-stream` / `output-stream` を得る）、`read`/`write` は `wasi:io/streams` の（blocking）read/write、`close` は resource の drop に対応する。ホストの listen 許可は wasmtime の `--tcplisten` 等で与える。
- **`emela run`（wasmi）**: 当面は `std::net`（`TcpListener` / `TcpStream`）でホスト実装してよい（現行 `http_host.rs` の socket 部分を `socket.*` に振り替える）。最終的には標準 `wasi:sockets` ホストへ寄せ、Emela 固有の Rust binding を無くす方向（プロジェクト方針）。
- **既存 registry**: `crates/emela-codegen/src/platform.rs` の `platform_interface()` に `socket.*` エントリ（capability `"Socket"`）を追加する。`http.server_*`（0046 旧）は 0046 改訂で撤廃される。
- **parity（0052）**: `socket.*` は `wasm-wasip2`（`wasi:sockets`）と `emela run`（wasmi の Rust ホスト）の双方で供給されるが、両者は本仕様の P 規則に**同一の観測挙動**で準拠しなければならない（0000 原則3・0052 T7/T8）。差分（conformance）テストで検証する。

### Open Questions

- **UDP・名前解決・接続（client 側 connect）**: 本仕様は listen/accept の server 側 TCP に絞った。`connect` や DNS、UDP は将来仕様。
- **タイムアウト・SO_* オプション・keep-alive**: ホストポリシーに留めるか、`Socket` の操作／設定値にするか。
- **`read` のストリーミング意味論**: chunk 単位読み取りとバックプレッシャ（wasi:io の pollable）を Emela 面にどこまで見せるか（0000 の async 非目標との兼ね合い）。

### 他仕様との関係

- **0009（Effect Semantics）**: 組み込み effect に `Socket` を追加する。予約されていた `net`（生ソケット）の TCP 部分を実体化する（`net` の残り＝UDP 等は予約のまま）。
- **0013（Platform Functions）**: registry に `socket.*` を追加する（供給契約・カバレッジ検査は不変）。
- **0043（Fallible Platform Functions）**: `socket.*`（`close` を除く）は `throws SocketError` を持つ fallible エントリである。
- **0051（Bytes）**: `Socket.read`/`write` の境界型 `Bytes` を定義する。
- **0049（Effects as Compile-Time DI）**: `Socket` はプリミティブ effect（leaf）。`HttpServer`（0046 改訂）はこの上の派生 effect であり、その handler が `uses { Socket }` を持つ。
- **0046（HTTP Server）**: `HttpServer` を `Socket` の上の派生 effect に再定義する（別途 0046 改訂）。`bind`/`accept`/`respond`/`close` の旧 host primitive は撤廃し、`Socket.*` + stdlib の HTTP パースへ置き換える。
- **0044（HTTP Client）**: `Http` は TLS がホスト責務ゆえ **`Socket` の上に置かず primitive のまま**（0044:224-225）。
- **0025（Capability Manifest）**: capability `"Socket"`・修飾名 `socket.*` が manifest に現れる。
