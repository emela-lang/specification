# 0025: Capability Manifest

Status: Draft

プログラムが要求する capability（platform 関数の集合，0013）と intrinsic の集合（0021）を，
生成物に**機械可読な manifest** として埋め込む仕様．ホストは**インスタンス化の前に**モジュールの
要求権限を監査できる．「effect row = サンドボックスの権限表」（0000）をツールチェーン全体の契約にする．

## Summary

- コンパイラは executable の**要求集合**（推移的に到達する platform 関数と intrinsic）を静的に確定する
  （0013 / 0021 のカバレッジ検査の入力と同一）．
- WASM ターゲットでは，この要求集合を custom section **`emela:capabilities`** に JSON として埋め込む
  (MUST)．
- manifest は計算された要求集合と**正確に一致**しなければならない（過少申告・過大申告の禁止, MUST）．
- エンコードは決定的（ソート済み・安定）とし，同一入力から同一バイト列を生成する (MUST)．
- ホスト（WAMR embedder，レジストリ，CI）は manifest を読み，供給可否・ポリシー適合を
  インスタンス化前に判定できる．

## Motivation

0013 により「プログラムが実行しうる effect 全体 = `main` から到達可能な platform 関数の集合」は静的に
確定する．しかしこの契約は現状コンパイル時にしか見えない．WASM モジュールを配布・デプロイ・埋め込む
場面では，**受け取る側**が「このモジュールは何を要求するか」を知る必要がある．

WASM の import 一覧からも要求はある程度読めるが，(1) capability 単位の分類（`io` / `fs` / …）が
失われている，(2) intrinsic の要求（0021, AOT 済みでも backend 供給前提の演算）は import に現れない，
(3) 監査ツールが Emela の命名規約に依存することになる．言語が規範化した manifest があれば，
「fs を要求するモジュールはこのデバイスに載せない」のようなポリシー検査を，実行系に依存せず書ける．

## Specification

### 要求集合の計算

- executable（`main` を持つ compilation unit）について，コンパイラは次を確定する (MUST)．
  - **requires**: `main` から推移的に到達可能な platform 関数の修飾名の集合（0013．embedder 定義の
    `host.*`（0026）を含む）．
  - **capabilities**: requires の各 platform 関数が属する capability の集合（0009 / 0026）．
  - **intrinsics**: 推移的に到達可能な intrinsic 名の集合（0021）．
- 集合は**実際の到達可能性**から計算する．関数の `uses` の過大宣言（0023）は要求集合を増やさない．

### 埋め込み

- WASM 出力では，manifest を名前 `emela:capabilities` の custom section に格納する (MUST for
  executables)．library（`main` を持たない unit, 0013）は同形式の metadata を保持してよい (MAY)．
- 非 WASM バックエンドは，生成物に同内容の manifest を添付すべきである (SHOULD)．担体（JS では併置
  JSON ファイル等）はバックエンドが定める（非規範）．

### 形式

manifest は UTF-8 の JSON で，次のフィールドを持つ．

```json
{
  "format": 1,
  "requires": ["io.write_stdout", "clock.now"],
  "capabilities": ["clock", "io"],
  "intrinsics": ["i32_add", "string_concat"],
  "entry": true
}
```

- `format`: manifest 形式のバージョン（整数）．本仕様は `1`．
- `requires`: platform 関数の修飾名の配列．**辞書順ソート・重複なし** (MUST)．
- `capabilities`: capability 識別子の配列．同上 (MUST)．
- `intrinsics`: intrinsic 名の配列．同上 (MUST)．
- `entry`: `main` を持つか．
- エンコードは決定的でなければならない (MUST): キー順は上記の通り，配列はソート済み，余分な空白を
  持たない．同一のプログラムからは同一のバイト列が生成される（reproducible build に資する）．

### 真実性

- manifest は計算された要求集合と正確に一致しなければならない (MUST)．要求するものを書き漏らすこと
  （過少申告）も，要求しないものを書くこと（過大申告）も許されない．
- 検証ツールは，モジュールの import 一覧・関数表と manifest の整合を検査してよく，不一致を拒否して
  よい (MAY)．

### ホスト側の利用（契約）

- ホストは manifest を読み，自身が供給する platform 関数の集合と比較して，`requires ⊆ 供給` でない
  モジュールのインスタンス化を拒否できる．これは 0013 のコンパイル時カバレッジ検査の**配布後の対**で
  ある．
- manifest による事前監査は，WASM の import 解決（未解決 import はロード失敗）を**置き換えない**．
  最終的な強制は常にサンドボックス側にある（0000 の二重保証）．

## Examples

```emela
import std.io.print

fn main() -> Unit uses { io } {
    print("Hello\n")
}
```

このプログラムの manifest:

```json
{"format":1,"requires":["io.write_stdout"],"capabilities":["io"],"intrinsics":["string_concat"],"entry":true}
```

WAMR embedder は instantiate 前に custom section を読み，「このデバイスでは `io` のみ許可」の
ポリシーに適合することを確認してから natives を登録・実行する．

## Compilation Notes

この節は非規範的である．

- 要求集合は 0013 / 0021 のカバレッジ検査で既に計算しているため，manifest 生成の追加コストはほぼ
  エンコードのみ．
- WAMR: `wasm_runtime_load` の前に custom section を読む軽量パーサをホスト SDK として提供すると良い．
- レジストリ / CI: `.wasm` から manifest を抜き出す CLI（`emela inspect` 等）を用意すると，デプロイ
  パイプラインでのポリシー検査が１コマンドになる．

## Open Questions

- capability ごとのより細かい記述（例 `fs` の read/write 区分）を format 2 で入れるか（0026 の粒度
  設計と連動）．
- 値マーシャリング ABI のバージョンを manifest に含めるか（0013 の Open Question と共有）．
- library の manifest を link 時に合成する規則（0013 の library metadata と統合）．
- 署名・改竄検出（manifest の真正性）はツールチェーンの責務か，仕様の責務か．
