## 0000: Language Design Principles

Status: Accepted

### 概要

Emela の仕様全体の設計原則を定義する．個別仕様（0001 以降）が衝突・逸脱したときの判断基準となる憲法である．

### 立ち位置

Emela は **「Gleam の書き味 × WASM ファースト × effect を型で追う」** 関数型言語である．
WAMR（WebAssembly Micro Runtime）を最も厳しい一次ターゲットとし，複数バックエンド（WASM/WAMR, JavaScript, Native）へコンパイルする．

中心テーゼ：

> **関数型の `uses` 節（effect row）が，そのまま WASM モジュールがホストへ要求する import 集合＝サンドボックスの権限表になる．**

これは二重の保証を与える．

- **静的保証**: 宣言なき副作用は effect 検査 (0009) が型エラーとして拒否する．
- **動的保証**: import されていない host 関数は，WASM サンドボックスが実行時にも呼ばせない．

「型検査器がサンドボックスポリシーを計算する言語」が Emela のアイデンティティである．

### 原則

1. **Effect は静的契約である**．effect row は型検査後に消去され，実行時表現を持たない（0022）．handler も辞書も evidence も渡さない．effect 機構のランタイムコストはゼロである．
2. **Capability ベースであって handler ベースではない**．Emela の effect は「要求の追跡と境界での供給」（0013）であり，Koka 型の effect handler（限定継続）は採らない．これは WASM の実行モデル（スタック切替なし）に合った effect の形を選んだという表明である．
3. **供給ベースのマルチバックエンド**．観測可能な意味はすべてのターゲットで同一でなければならない．バックエンドごとに違ってよいのは**何を供給するか**（platform 関数の部分集合 0013，intrinsic の部分集合 0021）だけである．コンパイラは「要求 ⊆ 供給」を静的に検査し，不足を列挙してエラーにする．
4. **コンパイラは小さな核，意味は2つの境界の外側に置く**．コンパイラが保持するのは値の表現と intrinsic → 命令表のみ（0021）．純粋演算の意味は stdlib（trait + intrinsic），副作用の意味は Runtime（platform 関数）が持つ．
5. **Progressive disclosure**．初学者は effect を知らずに書き始められる．private 関数の `uses` は推論され（0023），`pub fn` の境界でのみ明示を要求する．型機能（trait 0020，effect-row 多相 0022）は推論ファーストかつオプトインでなければならない．
6. **WAMR 制約を設計の強制関数にする**．wasm-gc に依存しない（0024），AOT 前提（単相化・静的解決），小フットプリント，決定的挙動．最も厳しいターゲットで動く設計は全ターゲットで動く．

### 3層モデル

- 表面構文は人間が書く契約である．
- IR はコンパイラが保証する意味である（0012）．
- Runtime Boundary は外界との副作用契約である（0013）．
- 構文糖衣は Core Semantics に影響してはならない．
- Native/WASM/WAMR で観測可能な意味が変わってはならない．

### 採らないもの

以下は明示的な非目標である．採用の提案は本仕様の supersede を要する．

- ownership / borrowing / move semantics
- effect handler / 限定継続（原則2）
- async（将来 capability として導入する余地は残す．現行設計をブロックしない）
- macro
- HKT / higher-rank polymorphism / GADT / dependent types
- trait object / 実行時ディスパッチ（0020 の「解決の限界」）
- wasm-gc への依存（0024）

trait と演算子の trait 化（0020），intrinsic（0021）は，初版の除外方針を更新して **opt-in の上級層** として採用済みである．

### 結論

Emela は次のモデルにする．

```text
strict expression-oriented functional core
+ immutable bindings by default
+ runtime-managed acyclic heap references   (0024)
+ effect row in function types, inferred    (0008, 0009, 0022, 0023)
+ capability-based runtime boundary         (0013, 0025, 0026)
```
