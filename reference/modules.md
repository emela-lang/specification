# Modules & Imports

> **リファレンス** — 現在の規範。経緯・動機は末尾 [Provenance](#provenance) の RFC を参照。

コードはモジュール単位で構成する。import は **モジュール単位** で行い、モジュール外の
関数呼び出しは常に修飾形になる。effect の宣言と呼び出しゲートは [Effects](effects.md) が、
trait の impl 可視性（孤児規則）は [Traits](traits.md) が定める。

キーワード MUST / MUST NOT / SHOULD / MAY は RFC 2119 に従う。

## 用語

| 用語 | 定義 |
|---|---|
| **モジュール** | 1ファイル。先頭の `module Name` ヘッダで名前を与える。 |
| **コンパイル単位** | コンパイルのルートとなるソース集合（entry）。 |
| **`pub`** | 公開。`pub` の無い定義はモジュール private。 |
| **import** | 他モジュールを取り込む宣言。対象は **モジュールのみ**。 |
| **修飾パス** | `list.map` / `std.list.map` のように `.` で繋いだ参照。 |
| **ベア名** | 1セグメントの名前（`map`）。 |
| **レシーバ呼び出し** | `x.f(...)`。値 `x` を先頭引数にする糖衣（→ [Traits](traits.md)）。 |

## 1. モジュールと可視性

- 1ファイル 1 モジュールである。ファイル先頭の `module Name` がモジュール名を与える。
- `pub` を持つ定義（関数・enum・record・effect）が公開アイテムである。`pub` の無い定義はモジュール private であり、importer からベア・修飾いずれの形でも参照できない (MUST NOT)。
- `pub fn` は型・effect を明示的に書く。`uses` の明示は必須である（→ [Effects の推論と注釈](effects.md#5-推論と注釈)）。

```emela
module user

pub fn get_user(id: UserId) -> Option<User> uses { Db } { ... }

fn decode(raw: String) -> User throws DecodeError { ... }   -- private
```

## 2. import（モジュール単位）

import の対象はモジュールのみである。関数・型・effect を個別に import する形式は存在しない (MUST NOT)。

- **R1（モジュール単位）**: `import p1.….pn`（n ≥ 1）はモジュールを import する。最終セグメント `pn` がモジュール名である。`p1` が依存パッケージ名なら[パッケージ](../specs/0032-packaging.md)ルート基準、そうでなければ import 元ファイル基準の相対パスで解決する。
- **R2（束縛）**: import は import 先モジュールの **公開アイテムのみ** を束縛する。
  - (a) **公開関数**: 修飾パスで参照可能にする。修飾子はモジュールパス `p1.….pn` の任意の非空 suffix である（`list.map(...)` / `std.list.map(...)`）。
  - (b) **公開型アイテム（enum / record / effect）**: ベア名で型スコープに入れる。
- **R3（ベア名の範囲）**: ベア名の関数参照は、ローカル束縛 → 同一コンパイル単位の関数の順で解決する (MUST)。import されたモジュールの関数に解決してはならない (MUST NOT)。診断は修飾形を案内する (SHOULD)。
- **R4（曖昧性は使用箇所で）**: 修飾パスが複数の import 先に一致する場合、および型アイテムのベア名が衝突する場合は、**使用箇所** で曖昧エラーとし、候補の full path を列挙する (MUST)。import 時にはエラーにしない (MUST NOT)。
- **R5（可視性）**: §1 のとおり、束縛（R2）の対象は公開アイテムのみである。
- [Core Prelude](runtime-boundary.md#5-core-prelude) のベア名は import を要さず常に参照できる。

## 3. 名前解決順位

パス `p1.….pk` の先頭 `p1` は次の順で解決する。

1. **ローカル束縛**（`let` / 引数）→ このとき `p1.x` はフィールドアクセスまたは[レシーバ呼び出し](traits.md#4-レシーバ構文)。
2. **大文字始まりならスコープ内の effect** → effect 操作呼び出し（→ [Effects の `uses` ゲート](effects.md#4-effect-操作の呼び出しと-uses-ゲート)）。
3. **import 済みモジュールの修飾子** → 修飾関数呼び出し（R2(a) の suffix 一致）。

enum variant・型変換は二重コロン `::` で書く型パスであり、`.` のパス解決とは構文的に分離される。`::` の左辺は enum 型名であり、`Enum.Variant` はエラーである。

```emela
import std.io
import std.list

fn main() -> Unit uses { Io } {
    Io.print(list.length([1, 2, 3]))   -- Io.print: effect 操作 / list.length: 修飾関数
}
```

## 4. backend シンボル

backend シンボルは所属モジュールで一律にマングルする（例：`std.list.map` → `std__list__map`、コンパイルルートはベア名のまま）。パス解決は型検査時に確定し、lowering は解決済みのシンボルを emit する。

## 未解決事項

- **alias**（`import std.list as l`）、および2つのモジュールが同名 effect / 型を公開する場合の衝突回避。現状は full path 修飾のみ。
- **再 export・推移的 import**（モジュールが import したものを外へ見せる手段）。現状は不可。
- 修飾された一次関数値（`let f = list.map`）の扱い。

## Provenance

- モジュール単位 import・束縛・可視性・名前解決順位（R1–R6）: [`../specs/0037`](../specs/0037-module-imports-and-first-class-effects.md)（[`0010`](../specs/0010-modules-imports-visibility.md) の import 節・[`0018`](../specs/0018-qualified-imports-and-calls.md) を置換）
- `module` ヘッダ・`pub` 可視性: [`../specs/0010`](../specs/0010-modules-imports-visibility.md)
