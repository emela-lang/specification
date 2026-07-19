## 0036: Effect Declarations and Effect-Qualified Operations

Status: Draft

`effect` 宣言を導入し、effect を「操作（operation）の集合を所有する第一級の実体」として定義する仕様．効果を使う側は effect 単位で import し、`uses { io }` を宣言した関数スコープ内で、その操作を修飾形 `io.print(...)` で呼び出す．（Flix の effect に着想を得るが、本仕様の範囲にハンドラ（再解釈）は含めない．）

### Summary

- `effect Name { ... }` は、名前付き effect と、それが所有する操作（`fn` / `extern fn`）の集合を宣言する．
- effect ブロック内の各操作は暗黙に `uses { Name }` を持つ．ブロック内で明示的な `uses` を書くことはできない．
- effect 宣言のファイルはモジュール `Name` を成す（effect 名 == モジュール名）．
- `import std.Name` は effect を丸ごと import する．その公開操作は **修飾形 `Name.op(...)` でのみ** 呼べる．ベア名 `op(...)` で effect 操作を呼ぶことはコンパイルエラーである．
- effect 操作の関数単位 import（`import std.Name.op`）は拒否する．
- `Name.op(...)` の呼び出しは、呼び出し元の推論 effect row に `Name` を寄与する．ゲートは既存の subset 規則（0023）が行う．
- 本バージョンにハンドラ（`try`/`with` による再解釈）は無い．操作の実体は 0013 の platform 関数 / extern が供給する．`effect` 構文は将来のハンドラの土台とする．

### Motivation

0013 までのモデルでは、`io` は「capability を発生させる platform 関数（`std.io.print` 等）を集めたモジュール」であり、利用側は関数を個別に import（`import std.io.print`）し、`uses { io }` を別途宣言していた．これは「`io` は関数の集まり」というモデルであって、「`io` という **効果（capability）** を使うことを宣言する」という直観と乖離していた．

本仕様は effect を第一級の実体に昇格させ、次を実現する．

1. 効果を使う側は effect 単位で import する（`import std.io`）．個々の操作を import する必要はない．
2. `uses { io }` を宣言した関数の中で、その effect の操作を `io.print(...)` の形で呼べる．どの effect の操作かが呼び出し地点で明示される．
3. 将来 ORM 等を実装した際、`effect db { ... }` を `uses { db }` の下で `db.query(...)` のように使える（実体は embedder-defined capability、0026）．

### Specification

以下、キーワードは RFC 2119 に従う．

#### effect 宣言

```
effect io {
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

- `effect Name { items }` はトップレベル宣言である．`Name` は単一の識別子でなければならない（MUST）．ドット付き名（`host.fs` 等）は本バージョンでは扱わない（Open Questions 参照）．
- `items` は `pub fn` / `fn` / `extern fn` を含んでよい（MAY）．`intrinsic fn` は effect の操作になれない（`intrinsic` は純粋でなければならない、0021）ので、effect ブロック内での `intrinsic` はエラーである（MUST NOT）．
- ブロック内の各操作は暗黙に `uses { Name }` を持つ（MUST）．操作に明示的な `uses` 節を書くことはエラーである（MUST NOT）．本バージョンでは、操作は自 effect 以外の effect を宣言できない．
- effect 宣言のファイルはモジュール `Name` を成す（MUST）．同一ファイルに `module` ヘッダと `effect` の両方がある場合、両者の名前は一致しなければならない（MUST）．1ファイルにつき effect は 1 つとする．
- `extern fn` の実体は 0013 の platform 関数として供給される．その canonical 名はモジュール名（== effect 名）で修飾される（例：`io.write_stdout`）．したがって platform capability の検証（0013）は従来どおり成立する．

#### import と可視性

- `import std.io` は effect `io` を丸ごと import する（MUST）．その公開操作は修飾子 `io`（および完全パス `std.io`）付きで参照できる．
- effect 操作の関数単位 import（`import std.io.print`）は拒否しなければならない（MUST）．診断は effect 単位 import（`import std.io`）を案内する．
- 非 effect のモジュール（`std.list`, `std.int` 等）の import は本仕様の影響を受けない．関数単位 import（`import std.list.map`）も従来どおり有効である．

#### 呼び出し（修飾必須）

- import された effect 操作は、修飾形 `io.print(...)` でのみ呼べる（MUST）．ベア名 `print(...)` は effect 操作に解決してはならない（MUST NOT）．ベア呼び出しは、修飾形を案内する診断とともにエラーとする．
- `io.print(...)` の呼び出しは、その操作の宣言 effect（`{ io }`）を呼び出し元の推論 effect row に union する．呼び出し元がその effect を `uses` に宣言していない場合、0023 の subset 規則によりエラーとなる（未処理 effect）．
- 補足：同一ファイル内で宣言された effect（import されていない）の操作は、修飾子（モジュールパス）を持たないため、そのファイル内ではベア名で参照される．修飾必須の制約は **import された** effect 操作にのみ適用される．

#### 例

正当：

```
import std.io

fn main() -> Unit uses { io } {
    io.print("Hello, Emela!\n")
}
```

エラー（ベア呼び出し）：

```
import std.io
fn main() -> Unit uses { io } {
    print("Hello\n")   -- error: `print` is an operation of effect `io`; call it as `io.print(...)`
}
```

エラー（操作の関数単位 import）：

```
import std.io.print    -- error: `print` is an operation of effect `io`; import the effect with `import std.io`
```

エラー（`uses` 欠落）：

```
import std.io
fn main() -> Unit {
    io.print("x\n")    -- error: unhandled effects — body uses ["io"] but `main` declares uses []
}
```

### Compilation Notes

（規範ではない．）本仕様は既存のモジュール／修飾名解決（0018）機構の薄い上層として実装できる．

- `effect Name { ... }` はパース時に、ordinary な `fn` / `extern fn` の集合＋`module = Name` へ desugar される．各操作には (a) 暗黙の `uses { Name }` と (b) 「effect 操作」マーカーが付く．
- 「effect 操作」マーカーを持つ関数は、**import された（非ローカルな）**場合、ベア名解決の候補から除外される．修飾パス（`io.print`）は従来どおり suffix 一致で解決する（0018）．
- effect 寄与・subset ゲートは既存の effect 推論（0023）がそのまま担う．新しい型規則は不要である．
- backend の emit 名・platform lowering は変わらない．effect の名前空間は呼び出し地点のパス解決にのみ影響し、backend シンボルには影響しない．

### 他仕様との関係

本仕様は次を精緻化／更新する．

- **0009（Effect Semantics）**：`io`, `clock` 等の組み込み effect は、操作を所有する effect である（本仕様が操作の宣言と呼び出しを規定する）．
- **0010（Modules, Imports, Visibility）**：effect の import 形式は `import std.io`（effect 単位）であり、effect 操作の関数単位 import は不可．
- **0013（Platform Functions）**：platform ラッパー（`print` 等）は `effect` ブロック内に置かれ、`write_stdout` 等は effect の（非公開の）操作となる．canonical platform 名・capability 検証は不変．
- **0018（Qualified Imports and Calls）**：effect 操作は、suffix のベア名解決が働かない唯一の例外である（修飾必須）．`.`（effect 操作）と `::`（enum variant／型変換）の区別は不変．
- **0023（Effect Inference and Subsumption）**：effect の寄与源が修飾 effect 操作の呼び出しになる．subsumption／subset の計算自体は不変．
- **0026（Embedder-Defined Capabilities）**：将来の `effect db { ... }` 等は embedder-defined capability をバッキングとする．

### Open Questions

1. **操作が追加の effect を宣言できるか**：本バージョンでは操作の effect row は厳密に `{ Name }`．`clock` も使う `io` 操作のような場合をどう扱うかは将来課題．
2. **同一ファイル内 effect の修飾呼び出し**：import されていない同一ファイルの effect 操作は、修飾子を持たないためベア名で呼ぶ．import された場合との一貫性（常に修飾必須にするか）は将来検討．
3. **未知 effect 名の検証**：`uses { X }` の `X` が既知（組み込み ∪ 宣言済み effect）でない場合の警告は、embedder-defined capability（0026）や `host.*` との兼ね合いから本バージョンでは導入しない（誤検出回避）．将来、closed set が定まった段階で lint 化を検討．
4. **`host.*` 等のドット付き effect 名**：本バージョンは effect 名を単一識別子に限定し、`host.*` は先送りする．
5. **ハンドラ**：`try`/`with` による操作の再解釈（代数的 effect ハンドラ）は本仕様の範囲外．`effect` 構文はその土台として設計されている．
