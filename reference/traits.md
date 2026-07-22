# Traits

> **リファレンス** — 現在の規範。経緯・動機は末尾 [Provenance](#provenance) の RFC を参照。

型が満たす **特性（trait）** を宣言し、その実装 `impl` を与え、ジェネリック関数に **境界（bound）**
として要求する。trait・impl・境界はすべて **単相化で静的に discharge** され、typed IR には
型変数・trait・辞書が一切現れない（[Effects の handler](effects.md#8-handlerdischargeerasure) と同じ erasure）。

キーワード MUST / MUST NOT / SHOULD / MAY は RFC 2119 に従う。

## 用語

| 用語 | 定義 |
|---|---|
| **trait** | 型が提供すべきメソッド（関連関数）のシグネチャの集合。 |
| **`Self`** | trait / impl の内側でのみ有効な予約型名。「この trait を実装する型」を指す。 |
| **impl** | `impl Trait for Type { ... }`。ある型が trait を満たすことと、その実装本体。 |
| **境界（bound）** | `<T: Trait>`。型パラメータに要求する trait 制約。 |
| **引数ディスパッチ** | `Self` がパラメータ型に現れるメソッド。実装を実引数の型から決める。 |
| **戻り値ディスパッチ** | `Self` が戻り値型にのみ現れるメソッド。実装を呼び出し位置の期待型から決める。 |
| **孤児規則（orphan rule）** | impl を書ける場所の制限。(trait, 型) ごとに実装を大域一意にする。 |
| **レシーバ構文** | `x.f(args)` ≡ `f(x, args)`。糖衣。 |

## 1. trait 宣言

```emela
trait Add {
    fn add(a: Self, b: Self) -> Self uses {}
}

trait Eq {
    fn eq(a: Self, b: Self) -> Bool uses {}
    fn neq(a: Self, b: Self) -> Bool uses {} { if eq(a, b) { false } else { true } }  -- default
}
```

- **`Self`（MUST）**: `Self` は trait / impl の内側でのみ有効な予約型名である。外で用いてはならない (MUST NOT)。
- **ディスパッチ可能性（MUST）**: 各メソッドは `Self` を **パラメータ型または戻り値型** の少なくとも一方（ネストを含む）に含まなければならない。
  - `Self` がいずれかのパラメータ型に現れる → **引数ディスパッチ**。実装は実引数の型から一意に決まる。
  - `Self` が戻り値型にのみ現れる → **戻り値ディスパッチ**（§7）。
  - `Self` がどこにも現れないメソッドは宣言できない (MUST NOT)。
- **`uses` / `throws`（MUST）**: trait メソッドは完全な関数型であり、[effect row（`uses`）](effects.md)と [error 型（`throws`）](errors.md)を宣言できる。ここで宣言した row はジェネリックコードが観測する契約である。
- **デフォルト実装（MAY）**: メソッドは本体を持ってよい。`impl` はデフォルトを持つメソッドの本体を省略してよい。

## 2. impl 宣言

```emela
impl Add for Vec2 {
    fn add(a: Vec2, b: Vec2) -> Vec2 uses {} {
        Vec2 { x: a.x + b.x, y: a.y + b.y }
    }
}
```

- **シグネチャ一致（MUST）**: impl 内の各メソッドは、trait 宣言に `Self := 対象型` を代入したものと **厳密に一致** しなければならない（引数型・戻り値型・`uses`・`throws` すべて）。impl 内では `Self` と対象型名のどちらで書いてもよい。
- **網羅性（MUST）**: デフォルトを持たない全メソッドに本体を与えなければならない。
- **余剰・重複（MUST NOT）**: trait に無いメソッドの定義・同一メソッドの二重定義は禁止。
- **対象型の形**: 名前付き型（`record` / `enum`）、組み込み型（`Int` / `Float` / `String` / `Char` / `Bool`）、型構築子に型パラメータを適用した形（`Array<T>`）。最後の形では impl 自身が型パラメータと境界を持てる（**パラメータ付き instance**、例 `impl<T: Show> Show for Array<T>`）。組み込み型への impl も書けるが孤児規則（§5）に従う。

## 3. 境界（bounds）

ジェネリック関数の型パラメータに `: Trait` の形で境界を付ける。

```emela
fn print_line<T: Show>(value: T) -> Unit uses { Io } {
    Io.print(to_string(value) ++ "\n")   -- to_string は Show、Self=T を引数から推論
}
```

- **構文**: `<T: Trait>`。複数境界は `+` で連結（`<T: Add + Show>`）。複数の型パラメータはそれぞれに境界を付けられる。
- **不透明性の精密な緩和（MUST）**: 境界 `T: Trait` の下では、`T` 型の値に対して汎用操作（束縛・受け渡し・戻り・`Array<T>`/`Option<T>` の出し入れ・関数なら適用）に加えて、`Trait` のメソッド呼び出しだけが許される。境界に無い操作（算術・比較・`match`・フィールドアクセス）はコンパイルエラーである。
- **メソッドの呼び出し形**: trait メソッドは自由関数名で裸に呼ぶ（`add(acc, x)`、`to_string(v)`）。`Self` に当たる具体型は実引数型から決まる。メソッド名が in-scope の複数 trait で衝突するときは曖昧エラーとし、`Trait.method`（2セグメントパス、[Modules の修飾パス](modules.md#3-名前解決順位)と整合）で明示解決する。
- **境界の伝播（MUST）**: 境界付き関数が別の境界付き関数を呼ぶとき、呼び先の境界は呼び元の境界から満たされていなければならない。
- **discharge（MUST）**: 推論が `T = C`（具体型）を定めた後、各境界について `impl Trait for C` が in-scope に存在するか検査する。無ければ「`C` は `Trait` を満たさない（実装が無いか import が漏れている）」エラーとする。

## 4. レシーバ構文

`recv.method(args)` は `method(recv, args)` への糖衣である（UFCS）。実装は `recv` の型から選ぶ。

- `recv.to_string()` ≡ `to_string(recv)`、`v.add(w)` ≡ `add(v, w)`。
- ドット左辺が **値（in-scope の束縛）** のときレシーバ呼び出しとなり、左辺がモジュール・trait 名のとき（`std.int.to_string`、`Show.to_string`）は[修飾パス](modules.md#3-名前解決順位)として解釈する——両者はドット左辺が値か否かで一意に区別できる。enum variant は `::` の型パスであり競合しない。
- 糖衣であって Core Semantics に影響しない。

## 5. コヒーレンス（孤児規則）

- **孤児規則（MUST）**: `impl Trait for Type` を書けるのは、**`Trait` を定義したモジュール**、または **`Type` を定義したモジュール** のいずれかに限る。どちらでもないモジュールでの impl はコンパイルエラー（孤児 impl）である。組み込み型の所有モジュールは std の該当モジュール（`Int` は `std.int` 等）と規定する。
- **大域一意**: 帰結として、(trait, 型) ごとに実装は高々1つになる。型が決まれば実装は一意で、instance 選択に曖昧は生じない。
- **可視性**: impl は必ず「trait のモジュール」か「型のモジュール」に住むので、**その trait とその型が両方 in-scope なら impl も自動的に in-scope** とみなす (SHOULD)。利用者は trait と型を import すれば、impl のための追加 import を要しない。
- **曖昧の報告（MUST）**: 曖昧が起こりうるのは **メソッド名の衝突**（in-scope の2つ以上の trait が同名メソッドを宣言）だけであり、[Modules の R4](modules.md#2-importモジュール単位) と同形式で報告し `Trait.method` で解決させる。実装が見つからない場合はこれと区別して「(trait, 型) の実装が in-scope に無い」エラーとする。

## 6. 演算子の trait 化

次の演算子は、対応する組み込み trait のメソッドへの糖衣である（規範）。

| 演算子 | trait | メソッド | シグネチャ |
|---|---|---|---|
| `+` | `Add` | `add` | `(Self, Self) -> Self` |
| `-` | `Sub` | `sub` | `(Self, Self) -> Self` |
| `*` | `Mul` | `mul` | `(Self, Self) -> Self` |
| `/` | `Div` | `div` | `(Self, Self) -> Self` |
| `%` | `Rem` | `rem` | `(Self, Self) -> Self` |
| `++` | `Concat` | `concat` | `(Self, Self) -> Self` |
| `==` | `Eq` | `eq` | `(Self, Self) -> Bool` |
| `<` | `Ord` | `lt` | `(Self, Self) -> Bool` |

- 式 `a ⊕ b` は対応する trait メソッド呼び出しへ desugar される（`a + b` ≡ `Add.add(a, b)`）。糖衣であって Core Semantics に影響しない。
- **組み込み型の instance**: `Int` / `Float` / `String` に対する演算子 trait の実装は、コンパイラが直接持つのではなく [Core Prelude](runtime-boundary.md#5-core-prelude) の Emela `impl` として与え、本体が対応する [`intrinsic fn`](runtime-boundary.md#4-intrinsic純粋演算) を呼ぶ。適用範囲は `Add`/`Sub`/`Mul`/`Div`: `Int`,`Float`／`Rem`: `Int`／`Concat`: `String`／`Eq`/`Ord`: `Int`,`Float`。
- **暗黙の可視性（MUST）**: 演算子 trait と上記組み込み instance は、明示 import なしに常に in-scope である（`1 + 2` が import 無しで動く後方互換）。これは Core Prelude の暗黙 import で実現する。
- ユーザー型に `impl Add for Vec2` を与えると、その型に `+` が使えるようになる。

## 7. 戻り値ディスパッチと `Monoid`

`Self` が戻り値型にのみ現れるメソッド（`fn empty() -> Self` 等）は、`Self` を **呼び出し位置の期待型** から解決する (MUST)。新構文は導入しない。

- **期待型の供給元**: 束縛の型注釈（`let x: Money = empty()`）、関数引数位置のパラメータ型、関数の戻り値型、`match` / `if` の他分岐の型、同一式中で `Self` を共有する隣接オペランドの型。
- **generic 文脈**: 期待型が境界付き型変数 `T`（`T: Monoid` が in-scope）のとき `Self := T` とし、境界で discharge 済みとみなす。具体化は単相化時に `T = C` が定まってから行う。
- **解決不能はエラー (MUST)**: 期待型から `Self` を一意に確定できなければコンパイルエラーとし、型注釈を促す。文脈皆無の `let x = empty()` はこれに該当する。

`Monoid` は Core Prelude が宣言する組み込み trait である。

```emela
trait Monoid {
    fn combine(a: Self, b: Self) -> Self uses {}   -- 引数ディスパッチ
    fn empty() -> Self uses {}                     -- 戻り値ディスパッチ
}
```

- 法則（結合律・単位元）は実装が満たすべきだが (SHOULD)、コンパイラは検査しない。
- `Monoid` は **いずれの演算子にも desugar しない** (MUST NOT)。総称関数の境界としてのみ用いる（`sum<T: Monoid>`）。演算子 trait（能力の層）と `Monoid`（代数構造の層）は意図的に直交である。
- **組み込み instance**: `String`（`combine = string_concat`, `empty = ""`）のみ提供する (MUST)。`Int` / `Float` は加法（単位元 `0`）・乗法（単位元 `1`）のどちらを正準とするか曖昧なため提供しない（→ [未解決事項](#未解決事項)）。数値を畳むには record / newtype に `impl Monoid` を与える。
- **HKT を導入しない**。`Monoid` は kind `*` の単一型 `Self` を抽象化する通常の trait である。

```emela
fn sum<T: Monoid>(xs: Array<T>) -> T uses {} {
    fold_combine(xs, 0, empty())     -- empty() の Self は戻り値位置 T（sum の戻り値）から解決
}
```

## 8. 解決の限界

境界は **単相化により静的に discharge できるもの** に限る。次は辞書渡し / 実行時ディスパッチを要するため扱わない。

- **trait object / 存在型**（`Array<dyn Show>`、実行時 vtable）: 構文を導入しない。
- **発散する多相再帰**（有限個の特殊化に展開できない）: エラー。
- **境界付きジェネリック関数の第一級化**（値として束縛・受け渡し）: 禁止（ジェネリック関数の第一級化禁止を継承）。

**erasure（IR 不変条件）**: trait / impl / 境界は IR 化の前に完全に消去される。typed IR には型変数・trait・辞書は現れず、backend は具体型の直接 call だけを受け取る。特殊化した trait メソッドは `Trait__Type__method`（例 `Add__Int__add`、`Monoid__Money__empty`）にマングルする。

## 未解決事項

- **関連型（associated types）**・**trait 継承 / スーパークラス**（`trait Ord: Eq`）・**多パラメータ trait**。
- **trait object / 存在型**（辞書渡し / vtable）。
- **derive**（`derive(Show)` による instance 自動生成）。
- **blanket / overlapping impl** を孤児規則の大域一意性を壊さずに許す方法。
- **effect / error 多相と subsumption**: trait メソッドの `uses` / `throws` に [row 変数](effects.md#7-effect-row-多相)を許すか、impl 側が trait 宣言より小さい effect / `throws Never` を持つ subsumption を許すか（現状は厳密一致）。
- **数値の Monoid**（加法 vs 乗法）、**一つの型に複数の Monoid**、**代数階層**（`Semigroup`/`Group`/`Ring`）。
- **明示型引数（turbofish）**: 期待型が無い文脈での `empty::<Money>()`。
- **分離コンパイル境界** での辞書渡しフォールバック。

## Provenance

- trait 宣言・impl・境界・レシーバ構文・孤児規則・演算子 trait 化・単相化・erasure: [`../specs/0020`](../specs/0020-traits.md)
- 戻り値ディスパッチ（期待型解決）と `Monoid`: [`../specs/0047`](../specs/0047-monoid.md)
- 組み込み instance を支える intrinsic と Core Prelude: [`../specs/0021`](../specs/0021-intrinsics.md)（→ [Runtime Boundary](runtime-boundary.md)）
