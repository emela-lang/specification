# 0026: Embedder-Defined Capabilities

Status: Draft

言語標準の platform 関数 registry（0013）を，**embedder（ホスト実装者）が `host.*` 名前空間で
拡張**できるようにする仕様．WAMR 埋め込みの実用（デバイス固有のセンサー・アクチュエータ・独自 API）を，
標準 capability と同じ規律（effect row・カバレッジ検査・manifest）の下で可能にする．

## Summary

- capability 名前空間 **`host.*`** を embedder 用に予約する．0009 の組み込み effect `host` は，単独の
  capability ではなく**この名前空間の親**として再解釈する（`uses { host.gpio }` のように用いる）．
- embedder は **host interface package**（`host.` 配下のモジュールに `extern fn` を宣言した Emela
  ソース群）をビルドに供給することで，registry を拡張する．
- 拡張された platform 関数は，標準のものと同一の扱いを受ける: effect row に現れ（0009），カバレッジ
  検査され（0013），manifest に記載される（0025）．
- host interface package なしのビルドで `host.*` を参照すればエラーになる．`host.*` を要求する
  executable は embedder 固有であることが manifest から機械的に判る．

## Motivation

0013 は「registry は言語仕様が定め，registry 外の `extern fn` はコンパイルエラー」と定めた．これは
Emela ソースの backend 非依存性を守るために正しいが，WAMR 埋め込みの現実と衝突する: 組込みホストの
価値は**デバイス固有の API**（GPIO，センサー，独自プロトコル）を WASM モジュールへ公開することにある．
これらを言語標準 registry に列挙することは原理的にできない．

必要なのは「標準 registry の外側だが，無法地帯ではない」拡張点である．embedder 拡張にも標準と同じ
規律（型付き宣言・capability 追跡・要求⊆供給の検査・manifest 記載）を課せば，effect システムの保証は
保たれたまま，移植不能性が**明示的**（`host.*` を使った時だけ・manifest に現れる）になる．

## Specification

### 名前空間の予約

- capability 識別子として `host.<ident>` 形を導入する（0009 の effect 集合の拡張）．`<ident>` は
  embedder が定める．
- 0009 の組み込み effect `host` は，単独では使用できない (MUST NOT)．常に `host.<cap>` の形で用いる
  （0009 を部分更新する）．
- 標準 registry（0013）のモジュールパスと `host.*` は互いに素である (MUST)．言語標準の platform 関数が
  `host.*` に置かれることはなく，embedder が標準名前空間（`io` 等）に platform 関数を追加することも
  できない (MUST NOT)．

### host interface package

- embedder は，**host interface package** をビルド入力として供給できる (MAY)．これは `host.` 配下の
  モジュールに `extern fn` を宣言する通常の Emela ソース群である．

```emela
-- host interface package: src/gpio.emel
module host.gpio

pub extern fn write(pin: Int, value: Bool) -> Unit uses { host.gpio }
pub extern fn read(pin: Int) -> Bool uses { host.gpio }
```

- `extern fn` の規則は 0013 と同一である: 本体を持たない (MUST NOT)．backend を名指しする構文は
  存在しない (MUST NOT)．
- 宣言する capability は，そのモジュールのパスに対応する `host.<ident>` でなければならない (MUST)
  （`module host.gpio` 内の extern fn は `uses { host.gpio }`）．
- registry 検査（0013）は「標準 registry ∪ 供給された host interface package の宣言」に対して行う．
  package が供給されないビルドで `host.*` の関数を参照すれば，通常の未定義エラーである．

### カバレッジ検査と manifest

- `host.*` の platform 関数は，カバレッジ検査（0013: 要求 ⊆ 供給）の対象である (MUST)．選択された
  target / embedder が供給しない `host.*` 関数を要求する executable は reject する．
- manifest（0025）の `requires` / `capabilities` に `host.*` を記載する (MUST)．これにより
  「このモジュールは embedder 固有 API を要求する＝この embedder でしか動かない」ことが配布物から
  機械的に判定できる．

### 移植性の位置づけ

- `host.*` を参照しない Emela ソースの backend 非依存性（0013）は本仕様によって変化しない．
- `host.*` を参照するソースは，その host interface を供給する embedder に対してのみ有効である．
  これは違反ではなく，manifest に明示される**宣言された特殊化**である．

## Examples

アプリケーションは標準 API と同じ書き味で embedder API を使う．

```emela
import host.gpio.write

fn blink() -> Unit uses { host.gpio, clock } {
    write(13, true)
    sleep_ms(500)
    write(13, false)
}
```

manifest（0025）:

```json
{"format":1,"requires":["clock.sleep_ms","host.gpio.write"],"capabilities":["clock","host.gpio"],"intrinsics":[],"entry":true}
```

このモジュールを `host.gpio` を供給しない汎用ランタイムに載せようとすると，コンパイル時（カバレッジ
検査）にも，配布後（manifest 監査・import 解決）にも拒否される．

## Compilation Notes

この節は非規範的である．

- **WAMR**: `host.*` の platform 関数は import module 名 `host`（または `host.gpio` 等）の import に
  lower し，embedder が `wasm_runtime_register_natives` で実体を登録する．標準 platform 関数
  （WASI 経由）と登録経路が分かれるだけで，機構は同じである．
- **テスト**: 供給はインスタンス化ごとに差し替えられる．`host.gpio` をインメモリ fake で供給する
  テストランタイムを作れば，effect handler なしで embedder API のモックが実現する（0013 の供給契約の
  帰結）．
- **配布**: host interface package は embedder SDK として配布し，バージョンを持たせるとよい．

## Open Questions

- capability の粒度の指針（`host.gpio` 1 個か，`host.gpio.read` / `host.gpio.write` を分けるか）．
  標準 capability（`fs` の read/write 区分）の粒度設計と共通の課題（0025 format 2 と連動）．
- host interface package のバージョニングと manifest への記録．
- 標準 registry への昇格プロセス（広く使われる host API を将来 `io` / `fs` 級に標準化する手順）．
- `host.*` の関数が failure を返す場合の `throws` 規約（0013 の「第一段階は Unit 戻りのみ」と共有）．
