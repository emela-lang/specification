# 0048: ARC — WASM バックエンドの決定的参照カウント実行

Status: Draft

0024 が定めた回収戦略（決定的 RC）を，WASM/WAMR ターゲットの**規範**として具体化する仕様．
コンパイラが typed IR（0012）上に retain / release を挿入する ARC（automatic reference
counting）方式を採り，ヒープの非巡回性（0024）により cycle collector なしで完全に回収する．
move・所有権・借用が表層言語に現れないこと（0000 / 0001 の非目標）は不変である．

## Summary

- WASM バックエンドが生成するモジュールは，到達不能になったヒープ値を**決定的に**回収する
  （リークしない）．現行実装（free を持たない bump allocator）の置き換えである．
- retain / release はコンパイラが typed IR 上に挿入する内部命令であり，その挿入・省略は
  観測可能な意味を一切変えない．表層言語に move / 所有権は現れない．
- 自己末尾再帰ループ（0045）は，各反復の作業集合が有界なら**反復回数によらず有界のメモリ**で
  回る．0046 のサーバーループを長時間稼働させるための前提である．
- 割り当て失敗（メモリ上限）は panic（0011）である．
- typed IR に RC 命令が追加されるが，ホスト GC を持つバックエンド（JS）は無視してよい．

## Motivation

現行の WASM バックエンドは free を持たない bump allocator であり，String 連結・Array の純コピー
操作（0007）・enum / record / closure の確保がすべて蓄積して，線形メモリの上限で trap する．
短命な CLI 実行では顕在化しないが，0045 / 0046 が規範例とする常駐ループ（HTTP サーバーの
accept ループ）では致命的で，回収なしには長時間稼働プログラムが書けない．

0024 は戦略（決定的 RC）と根拠となる不変条件（非巡回性）を定めたが，(1) 回収の実装保証の水準
（SHOULD 止まり），(2) IR 上の RC 命令の契約，(3) 長時間稼働に必要な資源挙動（末尾ループの
定数メモリ），(4) OOM の挙動は未規定だった．本仕様はこれらを WASM ターゲットの規範として固定し，
0024 の Open Questions のうち「incref / decref 省略最適化の指針」「OOM panic の規約」を閉じる．

先行例：Nim の ARC（スコープ駆動の挿入．循環回収は ORC として分離——Emela はヒープが非巡回
なので ORC に相当する機構自体が不要），Swift の ARC，Koka の Perceus（precise RC と所有権
渡し）．本仕様の方式はこれらの「コンパイル時挿入 + 決定的解放」の系譜に属する．

## Specification

以下，キーワードは RFC 2119 に従う．

### 回収保証（A1–A7）

- **A1（決定的回収）**: WASM バックエンドが生成するモジュールにおいて，ヒープ値（0001）が
  到達不能になったとき，その記憶域は**決定的に**回収されなければならない (MUST)．回収点は
  最後の参照が消滅する評価点（束縛のスコープ脱出・一時値の消費・引数 / 戻り値の受け渡し）で
  ある．0024 が SHOULD とした決定的回収を，本ターゲットでは MUST に引き上げる．
- **A2（観測不能）**: 回収はプログラムから観測不能でなければならない (MUST)．finalizer・
  デストラクタ・weak reference は存在しない (MUST NOT)（0024 再掲）．
- **A3（OOM は panic）**: 割り当ての失敗（メモリ上限到達，`memory.grow` の失敗）は panic
  （0011）でなければならない (MUST)．通常評価へは復帰しない．
- **A4（意味の不変）**: RC 命令の挿入・省略・移動は，評価順（0003——引数は左から右）・
  effect row（0022 / 0023）・`throws`（0011）を含む観測可能な意味を変えてはならない (MUST)．
  move・所有権・借用を表層言語に導入してはならない (MUST NOT)——0000 / 0001 の非目標は不変で
  あり，本仕様は supersede を要求しない．
- **A5（末尾ループの有界メモリ）**: 自己末尾呼び出し（0045 T2）による反復において，各反復で
  到達不能になった値はその反復のうちに回収されなければならず (MUST)，未実行の release 等の
  RC 上の義務が反復をまたいで累積してはならない (MUST NOT)．したがって各反復の作業集合が
  有界なら，ループ全体のメモリ使用量は反復回数によらず有界である．
- **A6（静的データ）**: 静的データ領域に置かれる値（string literal）は回収の対象にしては
  ならない (MUST NOT)．これらへのカウント操作は省略してよい (MAY)．
- **A7（エラー経路）**: エラーチャネルによる大域脱出（0011 の `throw` / `?`）で到達不能に
  なった値にも A1 が適用される (MUST)．捕捉されないエラーで関数を離れる場合も同様である．

### typed IR の RC 命令（A8–A10）

- **A8（命令の追加）**: typed IR（0012）に次の 2 命令を追加する．
  - `retain e` — `e` の値の参照カウントを +1 し，その値を結果とする．`e` の型がヒープ型で
    ない場合は挿入されない．
  - `release x in e` — 束縛 `x` の参照カウントを −1 し（0 になれば再帰的に回収し），続けて
    `e` を評価してその結果を返す．
  これらはコンパイラが挿入する内部命令であり，フロントエンド構文に対応物を持たない
  (MUST NOT——A4)．
- **A9（無視可能性）**: 回収は観測不能（A2）であるから，別の手段で A1 相当を満たすバックエンド
  （ホスト GC を使う JS，0024）は RC 命令を無視してよい (MAY)．既定では RC 命令は WASM
  バックエンドの内部でのみ挿入され，外部プラグインへ直列化される IR ストリームには現れない．
- **A10（省略の許可）**: 実装は，A4 を満たす限り RC 命令を省略・統合・移動してよい (MAY)．
  とくに束縛の**最後の使用**における `retain` の省略と，それと対になる `release` の除去
  （move と同型の，観測不能な最適化——0024 の言う「借用と同型の最適化」）は行うことが望ましい
  (SHOULD)．ただし A1 の決定的回収を弱めてはならない（省略は回収を**早める**方向にのみ働く）．

### 他仕様との関係

- **0024（Memory Model）**: 本仕様は 0024 を refine する（supersede しない）．非巡回不変条件・
  観測不能性・RC 戦略はそのまま引き継ぎ，WASM ターゲットに限って SHOULD（決定的回収）を MUST に
  引き上げ，OOM panic（A3）と省略指針（A10）を規範化する．
- **0012（Typed IR）**: A8 の 2 命令を IR 契約に追加する．バックエンドは無視してよい（A9）．
- **0045（Self Tail Calls）**: T2（スタック不消費）と A5（有界メモリ）が組で，0046 の常駐
  ループを実用にする．
- **0011（Error handling）**: A3（panic）・A7（throw / `?` 経路の回収）．
- **0001 / 0002 / 0006 / 0007 / 0028**: ヒープ値の集合（String / Array / Record / enum payload /
  closure environment）と immutable 性・非巡回性の根拠．意味論は不変．
- **0000（Language Design Principles）**: 非目標（表層の ownership / borrowing / move，wasm-gc
  依存）は不変（A4）．WAMR 制約（小フットプリント・決定的挙動，原則 6）に適合する．
- **0013 / 0043（Platform Functions）**: ホスト境界は値のコピー渡しであり，ホストへ渡った後の
  参照寿命の問題は生じない．ホストが linear memory を直接参照する規約は本仕様の範囲外（0025 /
  backend ABI）．

## Examples

回収点（A1）——0024 の例に回収時点の注記を加えたもの：

```emela
fn make() -> Array<Int> {
    let xs = [1, 2, 3]      -- 確保
    let ys = [4, 5, 6]      -- 確保
    ys                      -- 関数を出る時点で xs は到達不能 → ここで決定的に回収（A1）
}
```

末尾ループの有界メモリ（A5）——反復ごとの一時値はその反復内で回収される：

```emela
fn tick(n: Int) -> Unit uses { Io } {
    if n == 0 {
        ()
    } else {
        let msg = "tick " ++ "tock"   -- 反復ごとに確保される String
        Io.print(msg)
        tick(n - 1)                   -- 末尾位置（0045）．msg はここまでに回収（A5）
    }
}
```

エラー経路（A7）——throw で捨てられる途中結果も回収される：

```emela
fn parse(s: String) -> Int throws ParseError {
    let trimmed = trim(s)          -- 確保
    if string_length(trimmed) == 0 {
        throw ParseError.Empty     -- 大域脱出でも trimmed は回収される（A7）
    }
    to_int(trimmed)?
}
```

いずれの「回収される」も観測不能な実装保証（資源挙動）であって，プログラムの意味論では
ない（A2）．

## Compilation Notes

この節は非規範的である．現行 WASM バックエンドの参照実装の設計を記す．

- **所有権規約（owned-args）**: すべてのヒープ型式は所有された（+1 の）参照を結果とし，裸の
  変数読みだけが借用である．変数読みは**所有位置**（call / 自己末尾呼び出しの引数，enum payload・
  record フィールド・array 要素の構築，`throw` の値，`let` の右辺，関数・arm・branch の末尾値）
  でのみ `retain` に包む．intrinsic / platform 引数（`array_push` の要素引数を除き非エスケープ）・
  scrutinee・guard・条件は借用位置である．関数はヒープ型引数を +1 で受け取り，全出口で release
  する（callee-consumes）．この規約は 0045 の変換（呼び出し→ジャンプ）後も caller 側の release
  地点を要求しないため，A5 と整合する．
- **挿入パス**: lowering（monomorphization・0045 の tail-call 書き換え）後の typed IR に対する
  独立パスとして挿入する．`let` 束縛は継続の各末尾リーフで release し，`match` は scrutinee を
  一時束縛して payload 束縛を借用として読む（subject が arm より長生きするため retain 不要）．
  closure のキャプチャ格納・record / array 要素の取り出し（取り出し値の retain）はバックエンドが
  発行する．
- **ヘッダ**: ヒープブロックは `[drop_idx: i32][refcount: i32][payload...]` とし，オブジェクト
  ポインタは payload 先頭を指す（負オフセットヘッダ）．既存の payload レイアウト（0013 / 0017 の
  String `[len][utf8]` 等）とフィールドオフセットは不変で，8 バイト整列も保たれる．`drop_idx` は
  確保サイトが静的に知る形状の drop 関数（関数テーブルの index）であり，release が 0 に達したとき
  `call_indirect` で子参照の release と解放を行う．closure の drop は型からは静的に選べない
  （`Function` 型の値がどのラムダ環境かは実行時まで不明）ため本質的に動的であり，それ以外の型も
  同じ機構に載せて release を単一の共有関数にする（0024 の `[refcount][kind]` 素描の具体化．
  型ごとの静的 release 呼び分けより機構が一つ減り，生成コードも小さい）．ブロックサイズは
  ヘッダに持たず，各 drop 関数が自形状から計算して free に渡す．
- **静的データ判定**: retain / release は先頭で「ポインタ < ヒープ開始アドレス」を判定し，静的
  データ（string literal）なら即 return する（A6）．
- **アロケータ**: first-fit・分割ありの単一 intrusive free list + bump．空きが尽きたら
  `memory.grow`．上限（既定 16 MiB）到達・grow 失敗は `unreachable`（= panic，A3）．解放済み
  ブロックは先頭 2 ワードを `[size][next]` に流用する．coalescing は行わない（確保サイズが規則的．
  断片化が問題になれば size-class 化——0024 と同判断）．
- **エラー経路（A7）**: throw しうる評価点をまたいで生きる所有一時値は挿入パスが束縛に A-正規化
  し，catch 入口で「生存中の管理束縛を release してローカルを 0 クリア」する．release 済み
  ローカルの 0 クリアを規約とすることで，部分進行・末尾ループ再入に対して二重解放が起きない．
- **検証用エクスポート**: 生存バイト数 `live_bytes`・ヒープ上端 `heap_top` を wasm グローバル
  としてエクスポートする（非規範．テストが「プログラム終了時に `live_bytes == 0`」を検証する）．
- **JS バックエンド**: RC 命令を受け取った場合は素通しする（retain → 中身，release → 継続）．
- **省略（A10）の素描**: IR は単一代入なので last-use 解析が単純に効く．最後の使用の `retain`
  を消し，対応するパス末尾の `release` を打ち消す（move）．さらに単一使用の `let` の転送，分岐
  ごとの早期 release，将来は Perceus 風の in-place reuse・静的値 interning が同じ枠組みに載る．

## Open Questions

- OOM panic の診断メッセージ・フックの規約（0024 から引き継ぎ）．
- メモリ上限の設定面（ビルドオプション / マニフェスト 0025 との関係）．
- 深い再帰構造（長い List）の drop の再帰深度（反復化するか）．
- free list の coalescing / size-class 化の要否．
- 静的値の interning（ゼロキャプチャ closure・`None` 等）．
- Native バックエンドへの同一 RC ランタイムの適用（0024 Compilation Notes の共有）．
