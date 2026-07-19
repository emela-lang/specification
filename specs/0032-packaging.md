# 0032: Packaging and Dependency Resolution

Status: Draft

Emela の配布・依存単位 **Pome** と，その依存解決方式を定義する仕様．Pome は中央レジストリではなく，任意の
Git ホスティング上のリポジトリとして供給され，**source path**（import path）で識別される（Go に倣う分散
モデル）．複数 Pome を束ねる workspace **Bushel**，発見のためのオプショナルなインデックス **Orchard** も
定める．Emela の中核である「effect row = サンドボックスの権限表」（0000 / 0025）を，パッケージ流通の層まで
広げる．

## Summary

- 配布・依存の単位を **Pome** と呼ぶ（crate / gem 相当）．1つの Pome は1つ以上の module（0010）を含む．
- Pome は**任意の Git リポジトリ**として供給される．中央レジストリは持たない．Pome は canonical な
  **source path**（例 `github.com/emela-lang/stdlib`）で一意に識別される．
- 依存はホスト別の**省略スキーム**でも書ける（例 `github:emela-lang/stdlib`）．正規化すると source path になる．
- バージョンは**リポジトリの git tag**（semver，`v` prefix）で表す．中央のバージョン台帳を要しない．
- Pome の定義は `Pome.toml`（TOML）で宣言し，解決結果は `Pome.lock`（解決 tag と commit を固定）に記録する．
- パッケージ操作の CLI は `emela pome <verb>` 名前空間に置く（`add` / `remove` / `list` / `update` /
  `install` / `search`）．比喩的な動詞を避け，標準動詞で揃える．
- `emela pome add` は対象を取得し，0025 の要求集合を**ソースから計算して**提示できる．中央レジストリの
  自己申告に依存せず capability を監査できるのが，分散モデルにおける Emela の中核性質である．

## Motivation

Emela は module・import・可視性（0010 / 0018）を持つが，**プロジェクトをまたいでコードを共有・配布する
単位と手段**は未定義である（0010 の Open Question「package system をいつ入れるか」）．

配布モデルとして，Emela は中央レジストリを設けず，**Git ホスティング上のリポジトリを直接依存できる**分散
モデルを採る（Go modules に倣う）．中央レジストリは運用・信頼・単一障害点・name squatting の負担を伴う．
import path をそのまま source location にすれば，任意のホスト（GitHub / GitLab / self-hosted 等）へ対称に
配布でき，取得経路が透明になる．

この選択は Emela に固有の利点を持つ．0025 により，各生成物が要求する capability の集合は**ソースから静的に
計算できる**．したがって「このライブラリは何を要求するか」を知るのに，レジストリの自己申告を信用する必要が
ない．`emela pome add` は取得したソースから要求集合を直接計算して提示できる．「effect row = 権限表」（0000）
の検証可能性が中央集権を不要にし，分散取得と特に相性がよい．

名称は Emela（伊 *mela* ＝りんご）に因む果樹メタファーで統一する．配布物は木になる果実 **Pome**（りんご等の
ナシ状果を指す植物学名），その一山が **Bushel**（りんごの計量単位），発見のためのインデックスが **Orchard**
（果樹園）である．

## Specification

### 用語

- **Pome**: 配布と依存の単位．1つ以上の module（0010）を含む Git リポジトリで供給され，git tag（V1）で
  バージョン付けされる．`main` を持つ Pome を **entry pome**，持たないものを **library pome** と呼ぶ（0013 の
  executable / library 区分に対応）．
- **source path**: Pome の canonical 識別子．`host/path` 形（例 `github.com/emela-lang/stdlib`）．
- **Bushel**: 同一リポジトリで複数の Pome を一緒に開発する作業単位（workspace）．
- **Orchard**: Pome を発見（検索）するためのオプショナルなインデックス．**依存解決には必須でない**（下記 R4）．

### Pome の識別と省略スキーム

- **S1**: Pome の canonical 識別子は source path `host/path` である（スキーム・末尾 `.git` を除いた正規形）．
  `https://github.com/emela-lang/stdlib` と `github.com/emela-lang/stdlib` は同一 Pome を指す (MUST)．
- **S2**: ツールはホスト別の省略スキーム `<host-alias>:<path>` を受け付けてよい (MAY)．少なくとも `github:` を
  提供すべきである (SHOULD)．正規化規則の例:

  ```
  github:emela-lang/stdlib    ->  github.com/emela-lang/stdlib
  gitlab:acme/util            ->  gitlab.com/acme/util
  codeberg:acme/util          ->  codeberg.org/acme/util
  ```

- **S3**: `Pome.toml` と `Pome.lock` は常に canonical な source path で依存を記録する (MUST)．省略スキームは
  入力上の便宜であり，保存形式には現れない．

### バージョニングと解決

- **V1**: Pome のバージョンは，リポジトリの **git tag**（semver，`v` prefix．例 `v1.2.0`）で表す (MUST)．
  public API の非互換変更（effect row の変更を含む，0018）は major を上げる．
- **V2**: 依存は version requirement で指定する．最小として完全一致とキャレット範囲（`^1.2` =
  `>=1.2.0, <2.0.0`）を持つ．完全な文法は後続仕様で定める（Open Questions）．
- **V3**: 解決器は各依存について，制約を満たす最大の tag を選び，その tag が指す **commit** まで確定して
  `Pome.lock` に記録する (MUST)．解が無い場合は失敗する (MUST)．
- **V4**: tag の無い commit を直接指す指定（branch / commit SHA）を許してよい (MAY)．その場合の擬似バージョン
  表記は後続仕様で定める（Open Questions）．

### Pome 定義（`Pome.toml`）

- **F1**: Pome のルートに `Pome.toml` を置く (MUST)．形式は TOML とする．
- **F2**: `[pome]` テーブルは `name`（この Pome の canonical source path），`version`（semver 文字列，V1），
  `emela`（対応する言語仕様バージョン）を持つ (MUST)．任意で `module`（依存されたときの import ルート名，
  M2）を持ってよい (MAY)．省略時は source path の leaf を用いる．
- **F3**: `[dependencies]` テーブルは，依存する Pome の source path から version requirement（V2）への対応を
  持つ．キーは canonical source path とする（S3）．
- **F4**: `Pome.toml` は**ソース側の依存宣言**であり，「この Pome が何に依存するか」を表す．生成物へ埋め込む
  capability manifest（0025 の custom section `emela:capabilities`）とは別物である．後者は「この生成物が何を
  要求するか」を表し，`Pome.toml` からではなく**実際の到達可能性**から計算される（0025）．

### ロックファイル（`Pome.lock`）

- **F5**: entry pome は依存解決の結果を `Pome.lock` に固定する (MUST)．library pome は保持してよい (MAY)．
- **F6**: `Pome.lock` は各依存について，解決した source path・tag・commit SHA・content hash を記録する (MUST)．
  content hash は取得物の完全性検証（改竄検出）に用いる．
- **F7**: `Pome.lock` のエンコードは決定的とし，同一の入力（`Pome.toml` 群と各リポジトリの tag 状態）から同一の
  内容を生成する (MUST)．reproducible build（0025）と整合させる．
- **F8**: `Pome.lock` は解決器が生成・更新するものであり，手で編集する対象ではない (SHOULD NOT)．

### Workspace（`Bushel.toml`）

- **F9**: 複数の Pome を束ねる workspace のルートに `Bushel.toml` を置く．`members` に各 Pome のパスを列挙する．
- **F10**: Bushel は単一の `Pome.lock` を共有し，member 間で依存バージョンを一貫させる (SHOULD)．

### CLI

- **C1**: パッケージ操作は `emela pome <verb>` 名前空間の下に置く (SHOULD)．動詞は標準的な語に限る．

  ```
  emela pome add    <src>[@<req>]   -- 依存を取得し，Pome.toml / Pome.lock を更新
  emela pome remove <src>           -- 依存を Pome.toml / Pome.lock から削除
  emela pome list                   -- 解決済み依存ツリーを表示
  emela pome update [<src>]         -- version requirement の範囲内で更新（省略で全依存）
  emela pome install                -- Pome.lock に従って依存を取得
  emela pome search <query>         -- Orchard（オプショナル）を検索
  ```

  `<src>` は canonical source path または省略スキーム（S2）を受け付ける (MUST)．
- **C2**: プロジェクト生成とビルドは pome 名前空間の外に置く (SHOULD)．`emela new <name>` は新規 Pome の骨組み
  （`Pome.toml` を含む）を生成する．`emela build` / `test` / `run` は現在の Pome を対象とする．
- **C3**: `add` / `remove` は `Pome.toml` と `Pome.lock` の両方を整合させて更新する (MUST)．
- **C4**: 公開は，リポジトリに semver の git tag を打って push することで行う（V1）．中央への publish 手続きは
  存在しない．CLI は tag 打ちの補助を提供してよい (MAY)．

### Capability 連携（0025）

- **CAP1**: `emela pome add` は，対象 Pome とその推移的依存を取得し，0025 の要求集合（要求 platform 関数と
  capability）を**ソースから計算して**提示できる (SHOULD)．中央インデックスの自己申告に依存しない．
- **CAP2**: これにより「依存を入れると新たに `net` を要求するようになる」ことを導入時に監査できる．
- **CAP3**: capability 提示は 0025 の配布後監査（ホスト側の instantiate 前検査）を**置き換えない**．最終的な
  強制は常にサンドボックス側にある（0000 の二重保証）．
- **CAP4**: Orchard を提供する場合，索引化のために各 Pome の 0025 manifest を保持してよい (MAY)．ただし依存解決の
  真実は常に取得したソースにあり，インデックスの申告ではない（R4 と整合）．

### Pome と module の関係

- **M1**: Pome を依存に加えると，その Pome が公開する module を import できる（0010 / 0018）．
- **M2**: 依存 Pome の module を import するときの**ルート名**は，既定でその Pome の source path の leaf
  （`github.com/emela-lang/stdlib` なら `stdlib`）とする．Pome は `[pome].module`（F2）で別名を宣言してよく，
  宣言があればそれをルート名に用いる．これにより repository 名（source path）と import 上の名前を分離でき，
  例えば `github.com/emela-lang/stdlib` を `import std.io.print` として使える．source path と，そのルートの下に
  見える module 名前空間の詳細な対応は 0010 / 0018 と連動して定める（Open Questions）．

### 解決の非集権性

- **R4**: 依存解決は，各依存の source path が指すリポジトリからの取得だけで完結しなければならない (MUST)．
  Orchard の可用性に依存してはならない．Orchard は発見（検索）専用であり，ビルドの必須経路ではない．

## Examples

`Pome.toml`:

```toml
[pome]
name = "github.com/emela-lang/json"
version = "1.2.0"
emela = "0.1"

[dependencies]
"github.com/emela-lang/parser" = "^2.0"
"gitlab.com/acme/util"         = "^0.3"
```

省略スキームでの依存追加:

```console
$ emela new hello

$ emela pome add github:emela-lang/stdlib
  Fetched github.com/emela-lang/stdlib
  Resolved v1.4.0 (commit a1b2c3d)
  Updated Pome.toml, Pome.lock

$ emela pome list
  hello 0.1.0
  └── github.com/emela-lang/stdlib v1.4.0
```

capability 監査つきの追加（CAP1）:

```console
$ emela pome add github:acme/http
  Fetched github.com/acme/http @ v0.4.0
  Capabilities required by this pome and its dependencies (computed from source):
    net, clock
  Proceed? [y/N]
```

公開（git tag を push するだけ，C4）:

```console
$ git tag v0.1.0 && git push origin v0.1.0
  # これで github.com/you/hello を ^0.1 で依存できる
```

## Compilation Notes

この節は非規範的である．

- `emela pome add` の capability 提示は，取得したソースに対し 0013 / 0021 / 0025 のカバレッジ計算を走らせる
  だけで実現でき，専用の索引を必要としない．中央レジストリを介さずに「要求 capability」を監査できるのは，
  0025 が要求集合をソースから決定的に計算できることの直接の帰結である．
- `Pome.lock` の content hash（F6）は，取得物の完全性検証（go.sum 的な用途）に使う．0025 の manifest 真正性
  （署名・改竄検出）の Open Question と連動しうる．
- 取得のキャッシュ・プロキシ（オフラインビルド，company mirror）は実装が定めてよいが，解決の真実は常に
  source path のリポジトリにある（R4）．プロキシは透過なキャッシュに留める．
- Orchard を実装する場合，公開リポジトリを走査して 0025 manifest を索引化すれば，capability 単位の検索
  （例「`fs` を要求しない Pome だけ」）を提供できる．検索は発見の補助であり，依存解決とは独立である．

## Open Questions

- major version と source path の関係（Go の `/v2` 方式のように path へ major を含めるか，tag だけで区別するか）．
- tag の無い commit / branch を指す擬似バージョン表記（V4）の具体形．
- version requirement の完全な文法（tilde・caret・range・pre-release の扱い）．
- vanity import path（独自ドメインを実リポジトリへ解決する meta 情報）を持つか．
- import ルート名は M2 で定めた（既定は source path の leaf，`[pome].module` で上書き可）．残るのは，その
  ルートの下に見える module 名前空間の正確な対応規則（0010 / 0018 と統合）．
- 完全性検証（content hash）の正準化方法と，署名（0025 の真正性 Open Question）との統合．
- Bushel 内でのバージョン統一ポリシーと，member 間 path 依存の解決順序．
- `[pome].emela`（対応言語バージョン）の互換判定規則．
- Orchard インデックスの API・登録方法（発見専用として最小限に留めるか）．
