## 0037: Module-Unit Imports and First-Class Effects

Status: Accepted

頭字語の命名は先頭大文字のみ（`Io`）とする（2026-07-18 決定）．

import をモジュール単位に一本化し、effect をモジュールから独立した大文字始まりの第一級実体に昇格させる仕様．関数単位 import（0010/0018）を廃止し、モジュール外の関数呼び出しは常に修飾形とする．effect の操作は effect 名で修飾して `Io.print(...)` と呼び、`uses { Io }` を宣言したスコープ内でのみ解決される．0010 の import 節・0018・0036 を置き換える．

### Summary

- `import` の対象はモジュールのみである．`import std.list` はモジュール `list` の公開関数を修飾形 `list.map(...)` で、公開型アイテム（enum / record / effect）をベア名で利用可能にする．関数単位 import（`import std.list.map`）は廃止する．
- import されたモジュールの関数はベア名で呼べない．ベア名が指すのはローカル束縛と同一コンパイル単位の関数だけである．
- `effect Name { ... }` はモジュール内のアイテム（enum と同格）になる．`Name` は大文字で始まる．effect 名 == モジュール名の制約と 1 ファイル 1 effect の制約（0036）は廃止する．
- effect 操作は effect 名で修飾した `Name.op(...)` の形でのみ呼べる．ローカル宣言か import かを問わず一様に修飾必須である（0036 Open Question 2 の解決）．
- `Name.op(...)` は、語彙的に囲む関数の `uses` 節に `Name` が宣言されているときにのみ解決される（uses は effect 操作のスコープゲート）．effect row の subset 検査（0023）は従来どおり意味論上の最終ゲートとして残る．
- `uses { Name }` の各名前は、スコープ内の effect（同一ファイル宣言 ∪ import 済みモジュールの公開 effect）に解決されなければならない．未知の effect 名はエラーである（0036 Open Question 3 の解決）．
- import が束縛するのは公開（`pub`）アイテムのみである．非公開関数・effect の extern バッキング操作は、importer からいかなる形でも参照できない．

### Motivation

1. **二本立て import の非対称**．0018 の関数単位 import はベア名を束縛し、0036 の effect import は修飾必須と、粒度ごとに逆の規則を持っていた．effect 操作は「suffix ベア名解決が働かない唯一の例外」（0018 注）となり、例外規則が実装の歪みを生んだ．v0.4.0 で確認された破れ：`import std.io` した側で非公開 extern 操作 `write_stdout` がベア名で呼べてしまい（可視性と修飾必須の両方に違反）、逆に修飾形 `io.write_stdout` は解決されない．import をモジュール単位に一本化すると、「モジュール外の呼び出しは常に修飾形」が唯一の規則になり、effect の修飾必須は例外ではなく通常規則になる．
2. **effect == module（小文字）は直観と乖離**．effect は「関数の入れ物」ではなく capability の宣言であり、型と同様の大文字実体として書けるべきである．`uses { Io }` は能力の宣言として読め、将来の embedder 定義 effect（`effect Db`、0026）とも自然につながる．
3. **uses が語彙的なゲートになっていない**．「effect は宣言したスコープ内でのみ使える」という規則が effect row の subset 検査（0023）にのみ依存し、診断も「Unhandled effects」という row の言葉で報告されていた．uses を名前解決のゲートにすると、「`Io` はこの関数の uses に宣言されていない」という位置も語彙も正確なエラーになる．

### Specification

以下、キーワードは RFC 2119 に従う．

#### import（モジュール単位）

- **R1（モジュール単位）**: `import p1.….pn`（n ≥ 1）はモジュールを import する．最終セグメント pn がモジュール名である．p1 が依存パッケージ名（0032）ならパッケージルート基準、そうでなければ import 元ファイル基準の相対パスでモジュールファイルを解決する．関数・型・effect を個別に import する形式は存在しない（MUST NOT）．
- **R2（束縛）**: import は import 先モジュールの次を束縛する．
  - (a) 公開関数：修飾パスで参照可能にする．修飾子はモジュールパス `p1.….pn` の任意の非空 suffix である（`list.map(...)` / `std.list.map(...)`）．
  - (b) 公開型アイテム（enum / record / effect）：ベア名で型スコープに入れる．
- **R3（ベア名の範囲）**: ベア名（1 セグメント）の関数参照は、ローカル束縛 → 同一コンパイル単位の関数の順で解決する（MUST）．import されたモジュールの関数に解決してはならない（MUST NOT）．診断は修飾形（`list.map(...)`）を案内する（SHOULD）．
- **R4（曖昧性は使用箇所で）**: 修飾パスが複数の import 先に一致する場合、および型アイテムのベア名が衝突する場合は、使用箇所で曖昧エラーとする．エラーは候補の full path を列挙する（MUST）．import 時にはエラーにしない（MUST NOT）．
- **R5（可視性）**: 非公開アイテムは importer からベア・修飾いずれの形でも参照できない（MUST NOT）．束縛（R2）の対象は公開アイテムのみである．
- **R6（`.` の解決順位）**: パス `p1.….pk` の p1 は次の順で解決する：(1) ローカル束縛（このとき `p1.x` はフィールドアクセス／レシーバ呼び出し）、(2) 大文字始まりならスコープ内の effect（effect 操作呼び出し）、(3) import 済みモジュールの修飾子．enum variant・型変換は従来どおり `::`（0018 R7）であり `.` とは衝突しない．
- Core Prelude（0021）のベア名は import を要さず従来どおり参照できる（本仕様の影響を受けない）．

#### effect 宣言（アイテム化）

```
module io

effect Io {
    extern fn write_stdout(s: String) -> Unit
    extern fn write_stderr(s: String) -> Unit

    pub fn print<T: Show>(s: T) -> Unit {
        write_stdout(s.to_string())
    }
    pub fn eprint<T: Show>(s: T) -> Unit {
        write_stderr(s.to_string())
    }
}
```

- `effect Name { items }` はモジュール内のトップレベルアイテムである．`Name` は大文字で始まらなければならない（MUST）．モジュール名との一致制約・1 ファイル 1 effect の制約（0036）は廃止し、1 モジュールに複数の effect を宣言してよい（MAY）．
- `items` は `fn` / `pub fn` / `extern fn` を含んでよい（MAY）．**操作は自分の依存 effect を `uses` 節で宣言してよい**（0049 PE2b：インラインのデフォルト実装が他 effect を使える。例 `fn info(msg) uses { Io } { ... }`）——旧「暗黙 `uses { Name }`・明示 uses は禁止」を 0049 が緩和し 0036 Open Question 1 を解決する．`intrinsic` はエラー（MUST NOT、0021）．操作を**呼ぶ側**には effect 名 `Name` が寄与し（capability マーカー）、`Name` は discharge で操作の依存へ落ちる（0049 D1/D2）．
- `extern fn` の canonical platform 名は従来どおり**モジュール名**で修飾する（例：`io.write_stdout`、0013）．effect 名 `Io` は `uses` row・呼び出し構文・診断にのみ現れ、backend シンボル・platform capability 検証には影響しない（MUST）．
- extern バッキング操作は同じ effect ブロック内の操作からのみベア名で呼べる．ブロック外（同一モジュール内を含む）からは参照できない（MUST NOT）．`Name.extern_op(...)` の形の参照は「非公開操作」エラーとする（SHOULD、公開操作の列挙を案内してよい）．

#### effect の参照と uses ゲート

- effect 操作の呼び出しは `Name.op(...)` の形のみである（MUST）．同一ファイル宣言か import かを問わず一様であり、ベア名 `op(...)` で effect 操作に解決してはならない（MUST NOT）．
- `Name.op(...)` が解決されるのは次の両方が成り立つときである（MUST）：
  1. `Name` がスコープ内の effect（同一ファイル宣言 ∪ import 済み公開 effect）に解決できる．
  2. 語彙的に囲む最も内側の uses 保持スコープ（関数定義・関数リテラル）の宣言 `uses` row に `Name` が含まれる．
  条件 2 を満たさない場合は「effect `Name` はこの関数の `uses` に宣言されていない」エラーとし、`uses { Name }` の追加を案内する（SHOULD）．
- `uses { Name }` の各名前はスコープ内の effect に解決されなければならない（MUST）．解決できない場合は「unknown effect」エラーとし、その effect を宣言するモジュールが既知なら import を案内する（SHOULD）．
- effect row の推論・union・subset 検査（0022/0023）は不変であり、意味論上の最終ゲートとして残る．高階関数・関数値経由の effect 寄与は従来どおり row 検査が捕捉する．

#### 診断の階段

同じ「使えない」でも原因ごとに別のエラーを出す（MUST）：

```
fn main() -> Unit uses { Io } {
    Io.print("hi")     -- error: unknown effect `Io`（import が無い）— add `import std.io`
}
```

```
import std.io
fn main() -> Unit {
    Io.print("hi")     -- error: effect `Io` is not declared in `main`'s `uses` clause
}
```

```
import std.io
fn main() -> Unit uses { Io } {
    print("hi")        -- error: `print` is an operation of effect `Io`; call it as `Io.print(...)`
    write_stdout("x")  -- error: unknown name（非公開操作はベア名でも見えない）
    Io.write_stdout("x") -- error: `write_stdout` is a private operation of `Io`
}
```

```
import std.io.print    -- error: imports are module-unit — use `import std.io`
                       --（親パスがモジュールに解決でき、最終セグメントがその公開名なら
                       --  この形の診断を出す; SHOULD）
```

正当：

```
import std.io
import std.list

fn main() -> Unit uses { Io } {
    Io.print(list.length([1, 2, 3]))
}
```

### Compilation Notes

（規範ではない．）

- effect アイテムは「通常の fn 集合＋（暗黙 uses、effect 操作マーカー、所属 effect 名）」へ desugar できる点は 0036 と同じである．変わるのは修飾子が effect 名（`Io`）になることだけで、backend の emit 名・platform lowering は不変である．
- 現行実装の「import 先の宣言を root プログラムへクローンしてマージする」方式を続ける場合でも、束縛規則（R2/R5）は満たさなければならない：クローンするのは公開アイテムのみとし、非公開関数・extern を root の名前空間に混ぜない．モジュールを名前空間として保持する実装（各関数が所属モジュールを持ち、解決が束縛表を引く）を推奨する．v0.4.0 の可視性リークは全宣言クローン＋pub 限定の修飾子刻印の組み合わせに起因した．
- backend シンボルは所属モジュールで一律にマングルする（例：`std.list.map` → `std__list__map`、コンパイルルートはベア名のまま）．0018 の「衝突時のみマングル」は廃止する．
- uses ゲート（条件 2）は名前解決層の検査であり、型規則は増えない．row 検査（0023）と二重になるのは意図的で、前者は診断の質、後者は意味論の健全性を担う．

### 他仕様との関係

- **0010（Modules, Imports, Visibility）**：import 節を本仕様が置き換える．`module` ヘッダ・`pub` 可視性の意味は不変．
- **0018（Qualified Imports and Calls）**：廃止する．suffix 修飾の考え方は R2(a) に、`.`／`::` の分離（R7）は R6 に引き継ぐ．ベア名束縛・使用箇所での曖昧解決（R3〜R5）は import 単位がモジュールになったことで大幅に縮退する．
- **0036（Effect Declarations）**：effect ブロック・暗黙 uses・修飾必須・操作の個別 import 不可の骨子を引き継ぎ、effect == module（小文字名）を廃止する．0036 の Open Question 2（ローカル effect の修飾）・3（uses の未知名検証）は本仕様で解決する．
- **0009 / 0013 / 0025**：組み込み effect 名は `Io`・`Clock` に改める．platform 関数の canonical 名（`io.write_stdout` 等）とその capability 検証（0013）は不変．capability manifest（0025）が列挙する effect 名は大文字表記に追従する．
- **0026（Embedder-Defined Capabilities）**：embedder 定義 effect は `effect Db { ... }` の形で型と同じ命名規約に統一される．`uses` の未知名検証（本仕様）との整合は「embedder が effect 宣言を含むモジュールを提供し、利用側がそれを import する」ことで取る．
- **0022 / 0023**：row の要素名が大文字になる以外は不変．
- **0006 / receiver 呼び出し**：`x.y` の解決順位は R6 が定める（ローカル束縛が最優先）．

### Open Questions

1. **alias**：`import std.list as l`、および effect ベア名の衝突（2 つのモジュールが同名 effect を公開する場合）の回避手段．R4 の曖昧エラーで検出はできるが、回避は alias 導入まで full path 修飾のみ．
2. **ドット付き effect 名（`host.*`）**：0036 Open Question 4 の続き．本仕様でも effect 名は単一識別子に限定する．
3. **再 export・推移的 import**：モジュールが import したものを外へ見せる手段は未定義（現状どおり不可）．
4. **ハンドラ**：0049 が**静的 handler**（派生 effect の実装をコンパイル時に解決・使用箇所へ注入）を導入した．`try`/`with` による**実行時**再解釈（限定継続）は引き続き範囲外である．
