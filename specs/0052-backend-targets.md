## 0052: Backend Targets — wasm-unknown / wasm-wasip2 と platform 関数の供給差

Status: Draft

WASM 系のビルドターゲットを、ホスト import を持たない純コア **`wasm-unknown`** と、WASI 0.2（component model）準拠の **`wasm-wasip2`** の2つに分ける仕様。各バックエンドが宣言する **供給 platform 関数の集合**を差別化し、0013 のカバレッジ検査（要求 ⊆ 供給）に「この capability はこのターゲットに無い」を語らせる（0000 原則3：供給ベースのマルチバックエンド）。`emela run`（wasmi）は速い in-process 実行として残し、`wasm-wasip2` component は wasmtime 等で実行する。全ターゲット・全実行経路で**観測可能な意味は同一**でなければならない（0000 原則3）。旧 `wasm-wasi`（preview1＋独自 `emela_http`）は撤廃する。

### Summary

- ビルドターゲットを Rust の target triple に倣って分ける：
  - **`wasm-unknown`**（≈ `wasm32-unknown-unknown`）：ホスト import ゼロの純コア wasm。標準 capability の platform 関数を**供給しない**。
  - **`wasm-wasip2`**（≈ `wasm32-wasip2`）：WASI 0.2 の **component** を出力。`io.*` / `clock.*` / `socket.*` 等を標準 WASI インターフェースへ lower する。
- 各バックエンドは自らが供給する platform 関数の集合を宣言する。`wasm-unknown` は空（＋ intrinsic は inline）、`wasm-wasip2` は WASI 0.2 に写せるもの全て。カバレッジ検査（0013）が「要求 ⊆ 供給」を静的に検査し、不足を build 時にエラーにする（0049 D6 と同じ手触り）。
- **`emela run`**（wasmi・コア wasm interpreter）は残す：速い in-process 実行のため、`io.*` / `socket.*` 等を Rust ホストで供給する内部実行経路とする（当面は移行的に Rust binding を持つ）。`wasm-wasip2` component は wasmi では実行できないため、**wasmtime 経路**で実行する。
- **parity（0000 原則3）**：`emela run`（wasmi）と `wasm-wasip2`（wasmtime component）で観測可能な意味は同一でなければならない。HTTP 等のプロトコル処理は Emela 側（0046）ゆえ全経路で同一コードであり、差異が生じうるのは primitive `socket.*` のみである。両ホストは 0050 の P 規則に準拠しなければならず、差分（conformance）テストで検証する。

### Motivation

1. **供給ベースのマルチバックエンド（0000 原則3 / 0013）を正面から使う**。「何を供給するか」だけがターゲット間で異なり、観測可能な意味は同一、という設計をターゲット構成に反映する。Rust の `wasm32-unknown-unknown`（freestanding）と `wasm32-wasip2`（WASI 0.2）の分離が良い先例である。
2. **独自 `emela_http` の撤廃**。旧 `wasm-wasi` は preview1＋独自 import（`emela_http`）で、標準 WASI ランタイムで動かなかった。`wasm-wasip2` は標準 `wasi:sockets` 等のみを import し、生成物が wasmtime・component 系 PaaS で動く。
3. **純ターゲットの明示**。ホストを一切仮定しない純計算・ライブラリ・埋め込み用途のために、import ゼロの `wasm-unknown` を用意する。`uses {}` のプログラムだけが通り、capability を使うと build 時に reject される。
4. **`emela run` の存置**。開発の反復速度のため、pure-Rust の wasmi による速い in-process 実行を残す（C 案）。ただし wasmi は component を実行できないため、本番形（component）とは実行経路が分かれる。この分岐が観測可能な意味に漏れないことを parity 要件で保証する。

### Specification

以下、キーワードは RFC 2119 に従う。

#### バックエンドと出力形

- **T1（`wasm-unknown`）**: ホスト import を一切持たない**コア wasm モジュール**を出力する（MUST）。標準 capability（`Io`/`Clock`/`Socket`/…）の platform 関数を供給しない（MUST NOT）。intrinsic（0021）は従来どおり inline される。`main` は export された通常関数として提供し、`_start`/WASI は前提しない（実行はホストが export を呼ぶ）。`host.*`（0026）の embedder-defined import は出力してよい（MAY、embedder が配線する）。
- **T2（`wasm-wasip2`）**: WASI 0.2（component model）の **component** を出力する（MUST）。標準 capability の platform 関数を、対応する標準 WASI インターフェースへ lower する（`io.*`→`wasi:cli`、`clock.*`→`wasi:clocks`、`socket.*`→`wasi:sockets`、MUST）。独自 import（`emela_http` 等）を出力してはならない（MUST NOT）。command world（`wasi:cli/run`＝`main`）で `emela run` 相当の一括実行を提供する。
- **T3（供給集合の宣言）**: 各バックエンドは供給する platform 関数の集合を宣言する（0013 の backend 供給契約）。カバレッジ検査（0013）は「プログラムが（0049 discharge 後の leaf として）要求する platform 関数 ⊆ そのターゲットの供給集合」を検査し、不足を build エラーにする（MUST）。エラーは不足関数と、それを供給するターゲット（例 `wasm-wasip2`）を案内する（SHOULD）。

| platform 関数（leaf） | `wasm-unknown` | `wasm-wasip2` |
|---|---|---|
| `io.*`（write_stdout/stderr） | — | `wasi:cli/stdout`,`stderr` |
| `clock.monotonic_seconds` | — | `wasi:clocks/monotonic-clock` |
| `socket.*`（listen/accept/read/write/close） | — | `wasi:sockets` + `wasi:io/streams` |
| `host.*`（0026） | embedder が配線 | embedder が配線 |
| intrinsic（0021） | inline | inline |

- **T4（旧 `wasm-wasi` 撤廃）**: preview1＋独自 `emela_http` の旧 `wasm-wasi` バックエンドと `http.server_*`（0046 旧）を撤廃する（MUST）。preview1 準拠ターゲット（`wasm-wasip1`）を移行的に設けてよい（MAY、Open Questions）。

#### 実行経路と parity

- **T5（`emela run`）**: `emela run` は wasmi による in-process 実行として残す（MAY）。wasmi はコア wasm を実行するため、`emela run` はコアモジュールを対象とし、供給集合の platform 関数（`io.*`/`socket.*`/…）を**ホスト（Rust）で供給する**内部実行経路とする。この経路は当面 Emela 固有の Rust ホスト実装を持つ（移行的）。
- **T6（component 実行）**: `wasm-wasip2` の出力（component）は wasmi では実行できない。wasmtime 等の WASI 0.2 ランタイムで実行する（`emela run --wasmtime`／外部 `wasmtime run` 等、CLI 表面は実装が定める）。
- **T7（parity — 規範）**: 0000 原則3 により、**同一プログラムの観測可能な意味は、`emela run`（wasmi）と `wasm-wasip2`（wasmtime）で同一でなければならない**（MUST）。ターゲット間で異なってよいのは「何を供給するか」（供給集合）だけであり、供給された platform 関数の観測可能な挙動は同一でなければならない。
- **T8（parity の担保）**: primitive `socket.*` の挙動契約は 0050 の P 規則である。各実行経路のホスト（wasmi の Rust 実装・wasmtime の `wasi:sockets`）は 0050 P 規則に準拠しなければならない（MUST）。HTTP 等の上位プロトコル処理は Emela 側（0046 の handler・stdlib）ゆえ全経路で同一コードであり、差異の生じうる面は `socket.*` primitive に限定される。**差分（conformance）テスト**——同一プログラムを両経路で実行し観測結果の一致を主張する——を parity の検証手段とする（SHOULD）。

### Compilation Notes

この節は非規範的な実装上の補足である。

- **backend registry**: `driver.rs` の registry（`driver.rs:19-26`）に `wasm-unknown` / `wasm-wasip2` を登録する。`--backend` で選択し、既定・別名は実装が定める。`ArtifactKind`（`backend.rs:36-41`）に component 種を追加する（現状 `WasmBinary`/`WasmText` のみ）。
- **component 化（`wasm-wasip2`）**: コアモジュールを canonical ABI で WIT world に包む（`wasm-encoder`/`wasm-tools` 相当）。import は `wasi:sockets`/`wasi:cli`/`wasi:clocks`、command は `wasi:cli/run`（`main`）を export する。サーバー（0046）は `main` ループが `wasi:sockets` を回す command component であり、wasi:http `incoming-handler` の proxy world は用いない（0046 Motivation 2）。
- **`emela run`（wasmi）**: コアモジュール＋wasmi の Rust ホストで `io.*`/`socket.*` を供給する（現行 `run.rs`/`http_host.rs` の HTTP 部分は Emela 側 handler へ移り、ホストは `socket.*`（生 TCP）のみ供給に縮小する）。最終的に標準 `wasi:sockets` ホストへ寄せ、Emela 固有 Rust binding を無くす方向（プロジェクト方針）。
- **供給宣言**: `platform.rs` の各 `PlatformFn` に「どのターゲットが供給するか」を持たせる（または各 backend が供給名集合を返す）。カバレッジ検査は leaf（0049 discharge 後）に対して行う。

### Open Questions

- **`wasm-wasip1`**: preview1 準拠の移行的ターゲット（widely supported だが標準 socket が無い）を設けるか。設けるなら `socket.*` は供給せず、`io.*`/`clock.*` のみに絞る。
- **`emela run` の CLI 表面**: component 実行を `emela run --wasmtime` にするか、`emela serve`／別サブコマンドにするか。wasmtime をツールチェーンに同梱するか、外部依存にするか。
- **`emela run` の wasmtime 全面移行（A 案）**: 最終的に wasmi を廃し wasmtime に一本化して Rust binding を完全に消すか、wasmi の速い経路を恒久的に残すか。
- **parity テストの体系化**: 差分テストの標準スイート（`socket.*` の EOF・部分 read・close・並行受付など 0050 P 規則の網羅）をどこに置くか（brix／専用 harness）。
- **`wasm-unknown` の `host.*` 配線**: embedder-defined capability（0026）を `wasm-unknown` で使う場合の import 命名・供給契約。

### 他仕様との関係

- **0000（Language Design Principles）**: 原則3（供給ベースのマルチバックエンド・観測意味の同一）の直接適用。parity（T7）は原則3 の規範化である。
- **0013（Platform Functions）**: backend の供給契約・カバレッジ検査の具体化。Compilation Notes の「WASM backend は WASI のみ import」をターゲット別（unknown＝import ゼロ／wasip2＝WASI 0.2 component）に更新する（別途 0013 編集）。
- **0049（Effects as Compile-Time DI）**: カバレッジ検査・供給判定は discharge 後の leaf に対して行う。未解決 effect（D6）と「未供給 platform 関数」は同じ境界で表出する。
- **0050（Socket）**: `socket.*` の供給は `wasm-wasip2`＝`wasi:sockets`、`emela run`＝Rust ホスト。両者は 0050 P 規則に準拠する（T8）。
- **0046（HTTP Server）**: HTTP は Emela 側ゆえ全経路同一。`wasm-wasip2` で標準 WASI のみ import する目的地。
- **0026（Embedder-Defined Capabilities）**: `host.*` はどのターゲットでも embedder 配線。`wasm-unknown` は host.* の主戦場になりうる。
- **0025（Capability Manifest）**: manifest は leaf から計算（不変）。ターゲット選択には依存しない。
