## 0049: Effects as Compile-Time Dependency Injection (Static Handlers)

Status: Draft

effect を **プリミティブ（ホスト供給）** と **派生（Emela の handler が供給）** に区別し、派生 effect の実装（handler）を**コンパイル時に解決して使用箇所へ静的注入する**仕様。handler は実行時の値ではなく、trait の impl（0020）と同じく単相化で discharge され IR 化前に消去される。これにより Koka 型の限定継続を伴う実行時 effect handler を導入せずに、effect を「依存を注入される能力（DI）」として組み立てられるようにする。0000 の原則2 と 0037/0036 の「handler は範囲外」を更新する。

### Summary

- effect を2種に分ける。**プリミティブ effect** は `extern fn` 操作を持ち、実体を 0013 の platform 関数（ホスト）が供給する（`Io`・`Clock`・`Socket`(0050)・`Http`(0044) 等）。**派生 effect** は `extern fn` を持たない。各操作は effect ブロック内の**インラインのデフォルト実装**（本体付き `fn`、0037 が既に許す形の一般化）を持つか、本体の無い**抽象操作**（`handler` が供給する）である（`HttpServer`(0046 改訂) 等）。
- **デフォルト実装は effect ブロック内にインラインで書く**。`handler Name for E uses { R } { <op 実装> }` は**名前付きの派生バリアント**——デフォルトと異なる実装を依存 `R` の上で与える（DI・テスト用モック）。trait の `impl`（0020）に対応する。
- 使用箇所 `E.op(...)`（`E` が派生）は、スコープで供給された handler をコンパイル時に解決する：`with H { ... }`（キーワードは `with`。`provide` も可）で名前付き handler `H` が供給されていればそれを、無ければ effect のインラインデフォルトを選び、**直接 call へ単相化置換**する。`with H` の配下の関数は供給 handler について特殊化される（handler 多相の単相化）。handler・派生 effect・その dispatch は typed IR（0012）化の前に完全に消去される（辞書渡し・evidence 無し、0020 と同じ）。
- effect row の観点では、派生 `E` の操作呼び出しは呼び出し元の **source row** に `E` を寄与する。コンパイラはこれを、解決された handler の依存 `R`（`with H` で供給された `H` の deps、無ければインラインデフォルトの deps）へ **discharge** し、固定点まで反復して**プリミティブ effect の集合（leaf row）** へ落とす。**`with H { e }` は `e` の row 中の `E` を `R` で置換する**（＝供給された Effect の依存関係に落ちる）。**leaf row ＝ ホスト import 集合 ＝ 権限表**である（0000 中心テーゼの精緻化）。
- capability manifest（0025）は従来どおり到達 platform 呼び出しから計算され（`uses` 非依存）、discharge 後の leaf のみを列挙する。派生 effect は manifest に現れない（純 Emela の抽象だから）。
- **インラインデフォルトも extern も無い抽象操作を持つ effect は、handler が供給されなければ「未解決 effect」として row に蓄積**し、discharge されずに伝播する。`main` や 0013 のカバレッジ境界で leaf row に非プリミティブ（未解決）が残れば**自動的にエラー**（D6）。
- 実装は段階的：本バージョンは**デフォルト経路**（extern／インラインデフォルト／抽象＋D6）を規定し、これだけで `HttpServer` を `Socket` 上のデフォルト実装として書ける。**名前付き handler の供給（`with`/`provide`）と handler 多相単相化**は本仕様で定めるが後続実装に回す。

### Motivation

0037/0036 は effect を第一級実体（`effect Name { ... }`）に昇格させたが、operation の実体は常に 0013 の platform 関数（ホスト）が供給するモデルであり、handler（`try`/`with` による再解釈）は明示的に範囲外だった（0036 Open Question 5、0037 Open Question 4）。その結果、`HttpServer`（0046）のように「本質は `Socket` の上の HTTP プロトコル処理」である capability も、独立したホスト供給の primitive として platform 関数（`http.server_bind` 等）に lower せざるを得ず、標準 WASI に無い独自 import（`emela_http`）を生む原因になっていた。

本仕様は次を可能にする。

1. **能力を能力の上に組む**。`HttpServer` を「`Socket` を使って書かれた Emela の handler」として定義できる。ホストが供給するのは最小の primitive（`Socket`・`Io`）だけになり、生成物は標準 WASI の葉だけを import する（0013:130 と整合）。
2. **erasure を壊さない DI**。Emela の effect は「静的契約・実行時表現ゼロ」（0000 原則1、0022）である。本仕様の handler は Koka 型の限定継続ではなく、**trait の impl と同じ静的解決・単相化・消去**（0020）で実現する。実行時に辞書も evidence も渡らず、ランタイムコストは 0 のまま保たれる。ユーザーが確認した設計：「依存をコンパイル時に解決し、使用箇所へ handler を注入する」。
3. **DI の土台**。`effect` 構文は 0036 以来「将来の handler の土台」と位置づけられてきた（0036:15,127、0037:141）。本仕様がその positive な定義を与える。effect を「注入される実装を持つインターフェース」として扱う方向（Effect-TS の Layer 依存に相当）の第一歩であり、スコープ限定注入・完全な DI スレッディングは後続仕様が積む。

### Specification

以下、キーワードは RFC 2119 に従う。本仕様は 0037（first-class effects）と 0020（traits の解決・単相化・erasure）を前提とし、それらを拡張する。

#### プリミティブ effect と派生 effect

- **PE1（プリミティブ effect）**: `extern fn` 操作を1つ以上持つ effect を **プリミティブ effect** と呼ぶ。その `extern fn` の実体は 0013 の platform 関数がホストから供給する（本仕様は 0013 を変更しない）。プリミティブ effect は default 実装（0037 の `fn` 操作）を持ってよい（MAY）。
- **PE2（派生 effect）**: `extern fn` 操作を持たない effect を **派生 effect** と呼ぶ。派生 effect の各操作は本体を持たない**シグネチャ宣言**（`fn op(...) -> T throws E`、本体省略）であり、実体は handler（下記）が供給しなければならない（MUST）。
- **PE3（排他）**: 1つの effect はプリミティブか派生のいずれかである。同一 effect が `extern fn`（primitive）と handler バッキング（derived）を混在させることは本バージョンでは扱わない（MUST NOT、→ Open Questions）。
- **PE4（leaf）**: プリミティブ effect を **leaf** と呼ぶ。派生 effect は discharge（D2）により、最終的に leaf の集合へ還元される。

```
-- プリミティブ effect（0037 のまま。extern が platform 関数へ lower）
effect Io {
    extern fn write_stdout(s: String) -> Unit
    extern fn write_stderr(s: String) -> Unit
    pub fn print<T: Show>(s: T) -> Unit { write_stdout(s.to_string()) }
}

-- 派生 effect（操作はインラインのデフォルト実装を持つ。本体が使う Io が依存）
effect Log {
    pub fn info(msg: String) -> Unit uses { Io } { Io.write_stdout("[INFO] " ++ msg ++ "\n") }
    pub fn error(msg: String) -> Unit uses { Io } { Io.write_stderr("[ERROR] " ++ msg ++ "\n") }
}
```

#### デフォルト実装・名前付き handler・`with`/`provide`

- **PE2b（インラインデフォルト）**: 派生 effect の操作は effect ブロック内に**本体付き `fn`** としてデフォルト実装を持てる（MAY。0037 が既に許す形の一般化）。本体は他の effect を使ってよく（例 `uses { Io }`）、その依存が discharge の対象になる（D2）。
- **PE2c（抽象操作）**: 本体も `extern` も持たない操作は**抽象操作**である（MAY）。`with`/`provide`（H4）で handler が供給されなければ未解決（H6）になる。
- **H1（名前付き handler）**: `handler Name for E uses { R } { items }` は派生 effect `E` の**名前付きの実装（バリアント）** を宣言する。`items` は `E` の各操作に対応する `fn` 実装からなり、本体が使用する effect の集合が依存 row `R` である。trait の `impl`（0020）に対応するが `Name` で識別され、`with`/`provide`（H4）で明示的に供給される。
- **H2（網羅・一致）**: handler は `E` のすべての操作にちょうど1つの実装を与えなければならない（MUST）。各実装のシグネチャは `E` の対応する操作と**厳密に一致**しなければならない（引数型・戻り値型・`throws`。MUST）。`E` に存在しない操作を実装してはならない（MUST NOT）。
- **H3（依存の健全性）**: handler／インラインデフォルトの各実装本体が使用する effect は、宣言した依存 row に含まれていなければならない（0023 の subset 規則。MUST）。依存に派生 effect を含めてよく（MAY）、discharge は推移的に進む（D2）。
- **H4（供給＝`with`/`provide`）**: 名前付き handler は `with H { <expr> }`（キーワードは `with`。`provide H in <expr>` 形も可）で、`<expr>` の評価中に effect `E` の実装として供給する。`<expr>` 内で `E` を要求する呼び出しは `H` で解決され、`E` は `H` の依存 `R` に discharge される（D2）。`with` の配下の関数は供給 handler について**単相化**される（handler 多相の特殊化。0020 の単相化にもう1軸を足す）。**`with`/`provide` を使わない場合はインラインデフォルト（PE2b）が使われる**。
- **H5（handler は値でない）**: handler は第一級の値ではない（MUST NOT）。束縛・受け渡し・戻りはできない。解決の対象であって実行時表現を持たない（D3）。
- **H6（未解決 effect）**: 抽象操作（PE2c）を持ち、`with`/`provide` で handler も供給されない派生 effect を **未解決 effect** と呼ぶ。その操作呼び出し自体はエラーにしない（MUST NOT）。effect は discharge されずに row に蓄積し呼び出し元へ伝播する。エラーは境界（D6）で生じる。これにより実装（handler）より先に利用側を書ける。

`with` で名前付き handler を供給すると、**同じ `Log` 依存のコードを別の実装で走らせられる**（effect 多相 / DI）。例：`Log` を Console（`Io`）に書くか、構造化データとして `Fs` に書くか。`work` は `Log` にしか依存せず、供給される handler で振る舞いと leaf が変わる：

```
-- Log はインラインデフォルト（uses { Io }）を持つ。以下は差し替え可能なバリアント
handler ConsoleLog for Log uses { Io } {
    fn info(msg: String) -> Unit  { Io.write_stdout("[INFO] " ++ msg ++ "\n") }
    fn error(msg: String) -> Unit { Io.write_stderr("[ERROR] " ++ msg ++ "\n") }
}

handler FileLog for Log uses { Fs } {                 -- 構造化ログを Fs に追記
    fn info(msg: String) -> Unit  { Fs.append("app.log", "{\"level\":\"info\",\"msg\":\"" ++ msg ++ "\"}\n") }
    fn error(msg: String) -> Unit { Fs.append("app.log", "{\"level\":\"error\",\"msg\":\"" ++ msg ++ "\"}\n") }
}

fn work() -> Unit uses { Log } {   -- Log にしか依存しない（Io か Fs かは知らない）
    Log.info("started")
}

fn main() -> Unit uses { Io, Fs } {
    with ConsoleLog { work() }   -- ここでは Log -> { Io }（work は ConsoleLog で単相化）
    with FileLog { work() }      -- 同じ work が Log -> { Fs }（FileLog で単相化）
}
```

#### 解決・discharge・erasure

- **D1（使用箇所の解決）**: `E.op(...)`（`E` は派生 effect）は、スコープで解決された実装——`with H`（H4）で供給されていれば `H` の、無ければインラインデフォルト（PE2b）の——`op` を選び、その本体への**直接 call へ単相化置換**する（0020 の trait メソッド discharge と同じ機構）。`with` で供給された配下では、`E` を使う関数は供給 handler について特殊化される（handler 多相）。実装本体がさらに派生 effect を呼べば再帰的に解決する。到達可能な特殊化は有限でなければならない（0014/0020 の前提を継承。発散する場合はエラー）。
- **D2（effect row の discharge）**: effect row の計算（0023）において、派生 effect `E` の操作呼び出しは呼び出し元の **source row** に `E` を寄与する（0037 の uses ゲートは `E` に対して従来どおり働く）。コンパイラは、各派生 `E` を、解決された実装の依存 row `R`（`with H` で供給された `H` の deps、無ければインラインデフォルトの deps）で置換する **discharge** を固定点まで繰り返す。**`with H { e }` は `e` の row 中の `E` を `R` で置換する**。デフォルトも extern も無い抽象操作を持ち handler も供給されない派生 effect（未解決 effect、H6）は置換されずに row に残る。この固定点の結果を **leaf row** と呼ぶ。leaf row はプリミティブ effect と、未解決の派生 effect（residue）からなる。
- **D3（erasure）**: 派生 effect・handler・その解決は typed IR（0012）化の前に完全に消去される（MUST）。typed IR には handler も派生 effect も現れず、残るのはプリミティブ effect の platform 呼び出し（`IrExpr::Platform`）だけである。辞書渡し・evidence・実行時ディスパッチは生成しない（0020 の erasure 不変条件、0000 原則1、0022 と一致）。
- **D4（権限表 ＝ leaf row）**: 中心テーゼ（0000）「`uses` 節 ＝ ホスト import 集合 ＝ 権限表」を精緻化する。派生 effect の導入後、**権限表を成すのは discharge 後の leaf row のプリミティブな要素**である。source row（派生 effect を含みうる）は人間が読む intent の宣言であり、実際にホストへ要求する import 集合はプリミティブな leaf である。leaf row に非プリミティブな residue（未解決 effect）が残る場合の扱いは D6 が定める。
- **D5（manifest）**: capability manifest（0025）は従来どおり到達可能な platform 呼び出し（`IrExpr::Platform`）から計算される（`uses` 非依存）。discharge 後に残るのは leaf の platform 呼び出しだけなので、manifest は自動的に leaf のみを列挙する。派生 effect（`HttpServer` 等）は manifest に現れない（MUST NOT，純 Emela の抽象であり新たなホスト権限を生まないため）。
- **D6（エントリ境界の全解決）**: プログラムのエントリ（`main`）、および 0013 のカバレッジ検査が働く executable 境界では、その関数の **leaf row はプリミティブ effect のみからなっていなければならない**（MUST）。leaf row に未解決の派生 effect（residue、H6）が残る場合、それは「実装（handler）が無く、ランタイムも供給しない effect」であり **コンパイルエラー**とする（0013 のカバレッジ検査「要求 ⊆ 供給」を、discharge 後の leaf について検査する一般化）。エラーは未解決の派生 effect 名と、handler が期待されるモジュールを示す（SHOULD）。中間の関数は未解決 effect を row に持ってよい（MAY）——エラーは境界でのみ生じる。

#### 既存仕様の更新

- **U1（0000 原則2）**: 「Capability ベースであって handler ベースではない……Koka 型の effect handler（限定継続）は採らない」を次のように更新する。**静的解決・単相化・消去される handler は採用する**（本仕様）。**採らないのは Koka 型の限定継続を伴う実行時 effect handler**である。0000 原則1（effect の実行時表現ゼロ・ランタイムコスト0）は本仕様の handler が消去されることにより**保持される**。非目標リストの「effect handler / 限定継続」は「限定継続を伴う実行時 effect handler（Koka 型）」に改める。
- **U2（0037 / 0036）**: 0037 Open Question 4・0036 Open Question 5・0036:15,127 の「handler は範囲外」を本仕様が解消する。ただし解消するのは**静的 handler**のみであり、`try`/`with` による実行時再解釈は依然導入しない。
- **U3（0023）**: effect row の推論・subset 検査は、派生 effect を row 要素として扱ったうえで D2 の discharge を追加する。詳細は 0023 の改訂で定める（pub 境界の推論拡張と併せて扱う）。
- **U4（0013）**: 変更しない。プリミティブ effect の extern → platform 関数 → ホスト供給の経路は不変である。派生 effect は 0013 の registry に現れない。

### Examples

派生 effect `Log` をプリミティブ `Io` の上に定義し、利用する。`main` は `uses { Log }` と書けば足り、`Io` を直接書く必要はない。manifest には（discharge 後の）`Io` が現れる。

```
import std.io
import std.log

fn greet(name: String) -> Unit uses { Log } {
    Log.info("hello, " ++ name)
}

fn main() -> Unit uses { Log } {
    greet("emela")
    -- source row: { Log }
    -- discharge:  Log -> { Io }（インラインデフォルトの依存）
    -- leaf row:   { Io } ← 権限表・manifest はこれ
}
```

抽象操作（デフォルトも extern も無い）を持つ effect を、`with`/`provide` も無しに使うと、`main` で露見してエラーになる（D6）：

```
effect Db {
    pub fn query(sql: String) -> String   -- 抽象操作（本体なし）
}

fn main() -> Unit uses { Db, Io } {
    Io.print(Db.query("select 1"))
    -- source row: { Db, Io }
    -- discharge:  Db は抽象で handler 供給なし → 残る／Io は primitive
    -- leaf row:   { Db, Io } ← Db が非プリミティブとして残存
    -- => error: effect `Db` has no handler; provide one with `with <handler> { ... }`
}
```

`HttpServer`（0046 改訂）を `Socket`（0050）の上の派生 effect として定義する（抜粋・説明用）。accept ループは Emela 側が所有し、HTTP パースは操作のデフォルト実装（stdlib）が行う。デフォルトはインラインで書く：

```
effect HttpServer {
    pub fn accept(server: Server) -> Incoming throws HttpError uses { Socket } {
        let conn = Socket.accept(server.listener)   -- 生 TCP
        let request = parse_http_request(conn)      -- HTTP パースは stdlib
        Incoming { conn: conn, request: request }
    }
    -- bind / respond / close も同様に Socket の上のインラインデフォルトで実装
}
```

`main` から回す serve ループ（0045 の自己末尾呼び出し）は従来（0046）どおり書けるが、`uses { HttpServer }` は discharge されて leaf row は `{ Socket }` になる：

```
fn serve_loop(server: Server) -> Unit uses { HttpServer } {
    try {
        let incoming = HttpServer.accept(server)
        HttpServer.respond(incoming, handle(incoming.request))
    } catch { e -> () }
    serve_loop(server)
}
```

### Compilation Notes

この節は非規範的な実装上の補足である。

- **trait 機構への相乗り**: handler 解決は trait の bound discharge / 単相化（0020）とほぼ同型である。参照実装では既存の境界解決（`crates/emela/src/typecheck.rs` の bound discharge、および 0020 の特殊化キュー）を拡張し、「派生 effect の操作呼び出し → default handler の実装への直接 call」への置換を、trait メソッド → impl メソッドの置換と同じ段階（型検査／lowering）で行う。
- **row の二層化**: 現行実装は各関数本体の effect row を bottom-up の union で合成し（`TypedFunction.body_effects`）、宣言 row との subset を検査している。本仕様は「source row（派生を含む）」の上に **discharge 変換**（派生 effect を handler の依存 row で置換する固定点計算）を追加する。leaf row を IR/manifest 経路に渡し、source row を診断・pub 境界検査に使う。
- **manifest 不変**: capability manifest（`crates/emela-codegen/src/manifest.rs`）は到達 `IrExpr::Platform` から計算されており `uses` に依存しない。派生 effect は D3 により IR 化前に消去され platform 呼び出しへ還元されるため、manifest 側の変更は不要で、自動的に leaf のみを列挙する。
- **backend 不変**: backend は具体化された platform 呼び出しと直接 call だけを受け取る。handler 固有の IR ノード・実行時機構は追加しない（0020 と同じく専用ノード無し）。

### Open Questions

1. **handler 多相の単相化（`with`/`provide` の実装）**: 供給機構は本仕様（H4）で規定したが、**実装は後段**（本バージョンはデフォルト経路＝extern／インラインデフォルト／抽象＋D6 のみ実装）。呼び出しグラフ全体への handler スレッディングを、関数を供給 handler について特殊化する単相化（0020 の単相化にもう1軸）としてどこまで自動化するか。`with` のネスト時の解決順（最内優先）、同一 effect に複数 handler が in-scope なときの扱いを詰める。テスト用モックは Effect-TS の `layerTest` に相当（0044:226 / 0046:203）。
2. **状態を持つ handler**: コネクションプール等、handler が呼び出し間で状態を保持したい場合。erasure（D3）と両立させるには、状態を通常の値として明示的に持ち回るか、スコープ限定注入（1）に資源のライフサイクル（acquire/release）を載せる必要がある。
3. **async スケジューラを primitive effect とする案**: 並行性を `future<T>` として effect に統合する将来像（WASI 0.3 async）。スケジューラを `Socket` と同格の primitive effect に落とし、`Async` を派生 effect としてその上に静的解決すれば、erasure を壊さずに async×effect を載せられるか。
4. **mixed effect**: 同一 effect が一部の操作を extern（primitive）で、一部を handler（derived）で持つことの是非（PE3 で当面禁止）。
5. **名前付き handler の可視性と import**: `handler Name for E` をどのモジュールに置け、`with`/`provide` で使うために `Name` をどう import・名前解決するか。

### 他仕様との関係

- **0000（Language Design Principles）**: 原則2・非目標リストを U1 のとおり直接改訂する（本プロジェクトは Accepted 仕様も直接編集する方針）。原則1・中心テーゼは D4 のとおり精緻化する。
- **0020（Traits）**: handler 解決・単相化・erasure の機構を共有する。名前付き handler は「effect に対する（名前付きの）impl」に相当し、`with`/`provide` で選択される点が trait（型からの一意解決）と異なる。
- **0037 / 0036（Effect Declarations）**: `effect Name { ... }` を前提に、operation の実体供給に handler の経路を追加する。U2 のとおり「handler は範囲外」を静的 handler について解消する。
- **0022 / 0023（Effect-Row Polymorphism / Inference）**: erasure（0022）を保つ。row 計算に discharge（D2）を追加する（0023 改訂で規定）。
- **0013 / 0025（Platform Functions / Manifest）**: registry・manifest の仕組みは不変。派生 effect は registry・manifest に現れず、discharge 後のプリミティブ leaf のみがホスト境界・権限表に現れる。0013 のカバレッジ検査（要求 ⊆ 供給）は D6 で「discharge 後の leaf が全てプリミティブであること」へ一般化される。
- **0046（HTTP Server）**: `HttpServer` を `Socket`（0050）上の派生 effect に再定義する（別途 0046 改訂）。本仕様はその一般機構を与える。
- **0044（HTTP Client）**: `Http` は TLS がホスト責務（0044:224-225）ゆえ**プリミティブ effect のまま**である。`Socket` 導入後も派生化しない。
