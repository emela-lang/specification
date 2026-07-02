## 0013: Platform Functions and Runtime Provision

Status: Draft

副作用の実体（capability effect の解決）を Runtime が提供する境界を定義する仕様．

言語は **platform 関数** の標準インターフェース（名前・型・capability）を定める．Emela コードは `extern fn` でそれを参照し，実体は持たない．各 backend / target は，そのインターフェースの**部分集合**を実装する．これにより，副作用の解決は必ず Runtime が行い，かつ Emela ソースは backend 非依存になる．

### Summary

- capability effect (0009) を発生させる唯一の手段は **platform 関数の呼び出し**である．
- platform 関数の集合は本仕様が定める言語仕様であり，個々のプログラムが任意に定義するものではない．
- Emela コードは `extern fn` で platform 関数を宣言し，その実体は持たない．Runtime（backend）が実体を供給する．
- 各 target は platform 関数の部分集合のみ実装してよい．未提供の platform 関数を要求する executable は reject する．

### Motivation

0009 は「組み込み effect は Runtime が解決し，解決できない場合はコンパイルエラー」と定めるが，「どの関数が effect を発生させ，どう Runtime に解決されるか」は未規定だった．本仕様はその境界を規範化し，次を保証する．

1. コンパイラは副作用を一切実装しない．副作用の実体は常に Runtime にある．
2. Emela ソースは特定の backend を名指しできず，移植可能である (0000)．
3. プログラムが要求する Runtime 契約（必要な platform 関数の集合）が静的に確定する．

### Specification

#### Platform 関数は副作用の唯一の源

capability effect (0009 の `io`, `fs`, `clock`, `random`, `log`, `net`, `host`) を発生させる手段は platform 関数の呼び出しのみである．

コンパイラは capability effect を持つ組み込み演算を提供してはならない (MUST NOT)．`uses {}` の純粋な式・演算（算術，比較，束縛，関数呼び出し等）は capability effect を生まない．

#### 標準インターフェース (registry)

本仕様は platform 関数を列挙する．各エントリは次を持つ．

- 修飾名: モジュールパス＋名前（例 `io.write_stdout`）
- 引数型と戻り値型
- capability: 0009 の組み込み effect のいずれか

初期セット (MVP)．戻り値が `Unit` で，失敗を値で返さない最小集合から始める．

```text
io.write_stdout : (String) -> Unit uses { io }
io.write_stderr : (String) -> Unit uses { io }
```

将来，0017 の標準 capability に合わせて `clock.now`，`random.int`，`fs.read` 等を追加する（本仕様を supersede せず拡張する）．

#### extern 宣言

Emela コードは platform 関数を `extern fn` で宣言する．本体を持たない．

```emela
extern fn write_stdout(s: String) -> Unit uses { io }
```

- `extern fn` のシグネチャ（引数型・戻り値型・capability）は registry のエントリと一致しなければならない (MUST)．
- registry に存在しない platform 関数の `extern fn` 宣言はコンパイルエラーである (MUST)．
- `extern fn` に本体を書くことはできない (MUST NOT)．
- backend を名指し，または backend ごとに分岐する構文は存在しない (MUST NOT)．したがって，`extern fn` を含む Emela ソースは backend 非依存である．

慣習として，`extern fn` は標準ライブラリが宣言し，`pub fn` で包む (0010)．アプリケーションはラッパーを通じて利用し，`extern fn` を直接書く必要はない．

```emela
module io

extern fn write_stdout(s: String) -> Unit uses { io }

pub fn print(s: String) -> Unit uses { io } {
    write_stdout(s)
}
```

#### Runtime 供給契約

- 各 target / backend は registry の**部分集合**を実装してよい (MAY)．すべてを実装する必要はない．
- executable が，選択された target の提供しない platform 関数を（推移的に）要求する場合，コンパイルはエラーにしなければならない (MUST)．エラーは不足している platform 関数を明示すべきである (SHOULD)．（0017 の reject 規則と整合．）
- library（`main` を持たない compilation unit）は，要求する platform 関数の集合を metadata として保持してよい (MAY)．library 単体では Runtime 供給を要求しない．

#### 保証

- capability effect の発生源は platform 関数のみであるから，プログラムが実行しうる effect 全体は，`main` から推移的に到達可能な platform 関数の集合に等しい．これが Runtime が解決すべき契約である．
- platform 関数は Emela 側に実体を持たないから，その解決は必ず Runtime（backend）が行う．コンパイラは fallback 実装を出力してはならない (MUST NOT)．

#### error との分離

platform 関数の失敗は capability effect ではなく値で表す（0011）．capability（`io` など）は「外界への依存」を表し，失敗そのものではない．第一段階の registry は失敗を返さない（`Unit` を返す）platform 関数に限る．失敗を返す platform 関数は，`Result` / `Option` を戻り値とする形で後続仕様において追加する．

### Examples

標準ライブラリ（`std` パッケージ）が platform 関数を包む．

```emela
-- std: src/io.emel
module io

extern fn write_stdout(s: String) -> Unit uses { io }

pub fn print(s: String) -> Unit uses { io } {
    write_stdout(s)
}
```

アプリケーションはラッパーのみを使う．

```emela
import std.io.print

fn main() -> Unit uses { io } {
    print("Hello, Emela!\n")
}
```

`io` を提供しない target でこの executable をビルドするとコンパイルエラーになる．

### Compilation Notes

この節は非規範的な実装上の補足である．

- **IR**: platform 関数の呼び出しは，通常の関数呼び出しとは区別された専用ノードとして typed IR に保持する（0012，effect 注釈を保つ）．
- **JS backend**: backend が platform 関数の実体を提供する．既定ではランタイムオブジェクトを生成物に同梱する（例 `__rt["io.write_stdout"] = s => process.stdout.write(s)`）．ホストが実体を注入する形にしてもよい．
- **WASM/WAMR backend**: platform 関数は WASI / host import に lower する．ユーザーには WASI を直接見せない (0016)．文字列はランタイム管理の linear memory 表現で受け渡す (0007, 0015)．生成モジュールが import するのは WASI の関数のみとし，未提供 capability はロード時にも解決不能となる．
- **カバレッジ検査**: 各 backend は提供する platform 関数の集合を宣言し，コンパイラはプログラムの要求 ⊆ 提供 を検査する．

### Open Questions

- registry の初期セットの確定（`io` と `log` を分けるか，`io.write_stdout` を `io` と `log` のどちらの capability にするか，0017 と整合）．
- 値マーシャリング ABI をどこまで本仕様に固定し，どこから backend ABI に閉じるか．
- `extern fn` をアプリケーションでも宣言可能にするか，`std` パッケージに限定するか．
- 失敗を返す platform 関数（`Result` / `Option` 戻り）の導入時期（enum / generics 仕様への依存）．
- capability を link 時に提供するか，Runtime 起動時に提供するか（0016 の Open Question と共有）．
