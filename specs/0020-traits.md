# 0020: Traits

Status: Draft

型が満たす **特性 (trait)** を宣言し，その実装 `impl` を与え，ジェネリック関数に **境界 (bound)**
として要求できるようにする仕様．「加算可能な型」「`to_string` できる型」のような制約を表明し，
その制約下で総称的なコードを書けるようにする．spec 0014（Generic Functions）を拡張する．

## Summary

- `trait Name { ... }` で，型が満たすべき **メソッド（関連関数）のシグネチャの集合** を宣言する．
- `impl Trait for Type { ... }` で，ある型が trait を満たすことと，その実装本体を与える．
- ジェネリック関数の型パラメータに境界を付けられる: `fn f<T: Add>(...)`．境界下では，その型
  パラメータに対して境界 trait のメソッドを呼べる（0014 の完全な不透明性を精密に緩和する）．
- trait メソッドは **`Self` を含む自由関数シグネチャ** の集合であり，dispatch（どの実装を選ぶか）は
  **引数の型** から決まる．呼び出しは自由関数形 `f(x, ...)` を基本とし，レシーバ構文 `x.f(...)` は
  その **糖衣** として提供する（`x.f(args)` ≡ `f(x, args)`）．レシーバ構文でも実装選択は第一引数
  `x` の型から決まるのであって，レシーバに固有の意味論は無い．
- 演算子 `+ - * / % ++ == <` は，対応する組み込み trait のメソッドへの糖衣（desugar）とする．
  `Int` / `Float` / `String` などの組み込み型に対する instance は **標準ライブラリの Core Prelude
  （spec 0021）** が与え（本体は `intrinsic fn` を呼ぶ），その暗黙 import により `1 + 2` のような
  既存コードは import 無しで従来どおり動く（後方互換）．
- 実装の選択（instance の解決）は **孤児規則 (orphan rule)** により (trait, 型) ごとに大域一意で，
  0014 の **単相化 (monomorphization)** をそのまま拡張して静的に行う．typed IR（spec 0012）には
  型変数・trait・辞書（dictionary）は一切現れない．辞書渡しを要する機能（trait object 等）は本仕様
  では扱わない．
- 本仕様は 0014 の Open Questions が予告した「型パラメータの境界 (bounds) / trait」を実現し，
  spec 0000 の「最初のバージョンでは trait / operator overloading を入れない」方針を，trait と演算子の
  trait 化に関してのみ **部分的に更新 (partial supersede)** する．

## Motivation

0014 のジェネリック関数は型パラメータ `T` を **完全に不透明** に扱う．`T` の値に対して算術・比較・
`match`・フィールドアクセスはできず，束縛・受け渡し・戻り・配列化だけができる．これは
parametricity を保証する一方で，「要素を足し合わせる」「要素を文字列にする」といった，型に
**能力を要求する** 総称関数を書けない．

現状の言語では，そうした能力は個別にハードコードされている．

- 演算子 `+ - * / %`・`++`・`== <` は `BinaryOp`（`crates/emela-codegen/src/types.rs`）として，
  型ごとに typecheck / lowering / 各 backend で特別扱いされる（0016 / 0017）．
- `to_string` は `std.int`（`examples/stdlib/src/int.emel`）に **`Int` 専用** の純粋 Emela 実装が
  あるだけで，型ごとに振り分ける機構が無い．新しい型を足すたびに，各所で個別対応が要る．

ユーザーは「加算可能な集合」「print できる集合」のような **能力の抽象** を表明したい．本仕様は，
その抽象を `trait` として与え，`impl` で個々の型に実装を結び付け，ジェネリック関数の境界として
要求できるようにする．同時に，既存の演算子と `to_string` をこの機構の上に載せ替える経路を定める．

## Specification

本仕様は spec 0014 を拡張する．0014 の型引数推論・単相化・「各型パラメータは少なくとも 1 つの
パラメータ型に出現しなければならない」規則を継承し，境界と `Self` に一般化する．また spec 0000 の
trait / operator overloading 不採用方針を，本仕様が扱う範囲でのみ更新する（0000 のその他の原則
— ownership/borrowing 不採用，構文糖衣が Core Semantics に影響しない，ターゲット間で観測可能な
意味が変わらない等 — は引き続き有効である）．

### trait 宣言

`trait` 宣言は，型名（trait 名）と，その型が提供すべき **メソッドのシグネチャの集合** からなる．

```emela
trait Add {
    fn add(a: Self, b: Self) -> Self uses {}
}

trait Show {
    fn to_string(value: Self) -> String uses {}
}
```

- **`Self`（MUST）**: `Self` は，trait 宣言および `impl` 宣言の内側でのみ有効な予約型名であり，
  「この trait を実装する型」を指す．trait / impl の外で `Self` を用いてはならない (MUST NOT)．
- **ディスパッチ可能性（MUST）**: trait の各メソッドは，**いずれかのパラメータ型の中（ネストを
  含む）に `Self` が出現していなければならない**．これは 0014 の「各型パラメータは少なくとも 1 つの
  パラメータ型に現れる」規則の `Self` 版であり，呼び出しの実引数型から実装を一意に決められることを
  保証する．戻り値位置にのみ `Self` が現れるメソッド（例 `fn zero() -> Self`，`fn parse(s: String) ->
  Self`）は，引数から実装を決められないため本仕様では宣言できない (MUST NOT)（→ Open Questions）．
- **`uses` / `throws`（MUST）**: trait メソッドのシグネチャは完全な関数型なので，effect row (`uses`)
  と error 型 (`throws`) を宣言できる．ここで宣言した row は，ジェネリックコードが観測する契約である．
- **デフォルト実装（MAY）**: trait メソッドは本体（デフォルト実装）を持ってよい．デフォルト本体は，
  同じ trait の他のメソッドや `Self` 上の（当 trait が保証する）操作を用いてよい．`impl` は
  デフォルトを持つメソッドの本体を省略してよい (MAY)．

  ```emela
  trait Eq {
      fn eq(a: Self, b: Self) -> Bool uses {}

      -- デフォルト実装: eq から neq を導く
      fn neq(a: Self, b: Self) -> Bool uses {} {
          if eq(a, b) { false } else { true }
      }
  }
  ```

### impl 宣言

`impl Trait for Type { ... }` は，`Type` が `Trait` を満たすことと，その全メソッドの実装本体を与える．

```emela
record Vec2 {
    x: Int
    y: Int
}

impl Add for Vec2 {
    fn add(a: Vec2, b: Vec2) -> Vec2 uses {} {
        Vec2 {
            x: a.x + b.x
            y: a.y + b.y
        }
    }
}
```

- **シグネチャ一致（MUST）**: impl 内の各メソッドのシグネチャは，trait 宣言の対応するメソッドに
  `Self := 対象型` を代入したものと **厳密に一致** しなければならない（引数型・戻り値型・`uses`・
  `throws` のすべて）．impl 内では `Self` と対象型名のどちらで書いてもよい．
- **網羅性（MUST）**: trait のメソッドのうちデフォルト実装を持たないものすべてに，impl は本体を
  与えなければならない．
- **余剰・重複（MUST NOT）**: trait に存在しないメソッドを impl が定義してはならない．同一メソッドを
  二重に定義してはならない．
- **対象型の形**: 対象型は，名前付き型（`record` / `enum`），組み込み型（`Int` / `Float` /
  `String` / `Char` / `Bool`），または型構築子に型パラメータを適用した形（`Array<T>`）であってよい．
  最後の形では impl 自身が型パラメータと境界を持てる（**パラメータ付き instance**）．

  ```emela
  impl<T: Show> Show for Array<T> {
      fn to_string(xs: Array<T>) -> String uses {} {
          -- 要素ごとに to_string(x)（Show for T）を呼んで連結する
          ...
      }
  }
  ```

- **組み込み型への impl（許可）**: 組み込み型に対しても impl を書けるが，後述の孤児規則に従う．
  これが `to_string` を本機構へ載せ替える経路になる（Compilation Notes）．

### 境界 (bounds)

ジェネリック関数（0014）の型パラメータに，`: Trait` の形で境界を付けられる．

```emela
fn sum<T: Add>(xs: Array<T>) -> Option<T> uses {} {
    match Array.get(xs, 0) {
        None -> None
        Some(first) -> Some(fold_add(xs, 1, first))
    }
}

fn fold_add<T: Add>(xs: Array<T>, i: Int, acc: T) -> T uses {} {
    match Array.get(xs, i) {
        None -> acc
        Some(x) -> fold_add(xs, i + 1, acc + x)   -- `+` は Add.add（後述）
    }
}
```

- **構文**: 型パラメータリスト内で `<T: Trait>` と書く．複数の境界は `+` で連結する
  (`<T: Add + Show>`)．複数の型パラメータはそれぞれに境界を付けられる (`<T: Show, U: Show>`)．
  境界結合子 `+` は `< >` の内側にのみ現れるため，算術演算子 `+` との構文的曖昧は生じない．
- **不透明性の精密な緩和（MUST）**: 境界 `T: Trait` の下では，`T` 型の値に対して，0014 が許す汎用
  操作（束縛・受け渡し・戻り・`Array<T>` の要素化・`Option<T>` への出し入れ・関数型なら適用）に
  加えて，**`Trait` のメソッドの呼び出し** だけが許される．複数境界 `T: Add + Show` の下では，
  それら全 trait のメソッドが使える．境界に無い操作（算術・比較・`match`・フィールドアクセス）は
  依然としてコンパイルエラーである．
- **メソッドの呼び出し形**: trait メソッドは自由関数名で裸に呼ぶ（`add(acc, x)`，`to_string(v)`）．
  `Self` に当たる具体型は，0014 の型引数推論により実引数型から決まる．メソッド名が in-scope の
  複数の trait で衝突するときは曖昧エラーとし，`Trait.method`（2 セグメントパス，spec 0018 R1 と
  整合）で明示解決する（例 `Show.to_string(v)`）．
- **レシーバ構文（糖衣, MAY）**: `recv.method(args)` は `method(recv, args)` への糖衣であり（UFCS），
  `recv` の型から実装を選ぶ．`recv.to_string()` は `to_string(recv)` と，`v.add(w)` は `add(v, w)` と
  同義である．ドット記法の先頭が **値（in-scope の束縛）** のときにレシーバ呼び出しとなり，先頭が
  モジュール・trait 名のとき（`std.int.to_string`，`Show.to_string`）は従来どおり
  修飾パス（0018）として解釈する — 両者はドット左辺が値か否かで一意に区別できる．なお enum variant
  は `::` の型パス（`Enum::Variant`，0005/0018 R7）であり，そもそもドット記法とは別系統なので
  レシーバ構文と競合しない．糖衣であって Core Semantics に影響しない（0000）．
- **境界の伝播（MUST）**: 境界付きジェネリック関数が別の境界付きジェネリック関数を呼ぶとき，
  呼び先の境界は呼び元の境界から満たされていなければならない（上例の `sum` → `fold_add`，いずれも
  `T: Add`）．
- **境界の充足検査 / discharge（MUST）**: 呼び出しにおいて 0014 の推論が `T = C`（具体型）を定めた
  後，コンパイラは各境界について **`impl Trait for C` が in-scope に存在するか** を検査する．
  存在すれば解決し，存在しなければコンパイルエラーとする（「`C` は `Trait` を満たさない．実装が
  無いか，実装を提供するモジュールの import が漏れている」）．

### コヒーレンス（孤児規則）

同一の (trait, 型) に対して複数の実装が衝突すると，同じ型が文脈により別の振る舞いをする
（incoherence）．本仕様はこれを **定義時に構造的に禁止** する．

- **孤児規則（MUST）**: `impl Trait for Type` を書けるのは，**`Trait` を定義したモジュール**，または
  **`Type` を定義したモジュール** のいずれかに限る．どちらでもないモジュールでの impl はコンパイル
  エラー（孤児 impl）とする．組み込み型（`Int` など）の所有モジュールは std の該当モジュール
  （`Int` は `std.int` 等）と規定する．
- **大域一意**: 孤児規則の帰結として，プログラム全体で (trait, 型) ごとに実装は高々 1 つになる．
  したがって **instance の選択に曖昧は生じない**（型が決まれば実装は一意）．
- **可視性**: impl を使う（境界を discharge する）には，その impl を提供するモジュールが，具体型が
  確定する地点（＝呼び出し側）で可視でなければならない．孤児規則より impl は必ず「trait のモジュール」
  か「型のモジュール」に住むので，**その trait とその型が両方 in-scope なら，その impl も自動的に
  in-scope とみなす**（SHOULD）．利用者は trait と型を import すれば，impl のために追加 import を
  する必要はない．
- **曖昧の報告（MUST）**: 上記より instance 選択自体は曖昧にならない．曖昧が起こりうるのは **メソッド
  名の衝突**（in-scope の 2 つ以上の trait が同名メソッドを宣言している）だけであり，これは spec 0018
  R5 と同じ形式（候補それぞれの `Trait` と所在モジュールを列挙）で報告し，`Trait.method` で解決させる．
  実装が見つからない場合は，これとは区別して「(trait, 型) の実装が in-scope に無い」エラーとする．

### 演算子の trait 化

次の演算子は，対応する組み込み trait のメソッドへの糖衣とする（規範）．

| 演算子 | trait  | メソッド | シグネチャ                  |
| ------ | ------ | -------- | --------------------------- |
| `+`    | `Add`  | `add`    | `(Self, Self) -> Self`      |
| `-`    | `Sub`  | `sub`    | `(Self, Self) -> Self`      |
| `*`    | `Mul`  | `mul`    | `(Self, Self) -> Self`      |
| `/`    | `Div`  | `div`    | `(Self, Self) -> Self`      |
| `%`    | `Rem`  | `rem`    | `(Self, Self) -> Self`      |
| `++`   | `Concat` | `concat` | `(Self, Self) -> Self`    |
| `==`   | `Eq`   | `eq`     | `(Self, Self) -> Bool`      |
| `<`    | `Ord`  | `lt`     | `(Self, Self) -> Bool`      |

- **desugar（規範）**: 式 `a ⊕ b`（`⊕` は上表の演算子）は，対応する trait メソッド呼び出しへ
  desugar される．オペランドの型に対してそのメソッドを discharge した結果が式の型・意味である．
  例: `a + b` は `Add.add(a, b)` と同義．糖衣は Core Semantics に影響しない（0000）．
- **組み込み型の instance（Core Prelude / spec 0021）**: 組み込み型に対する演算子 trait の実装は，
  コンパイラが直接持つのではなく，**標準ライブラリの Core Prelude（spec 0021）の Emela `impl`** として
  与える．各 `impl` の本体は，対応する `intrinsic fn`（spec 0021）を呼び，backend が native 命令へ
  インライン lower する．この instance の集合は，spec 0016 / 0017 が定める現行の演算子の型付けと
  **完全に一致** させる（後方互換のため）．すなわち，

  - `Add` / `Sub` / `Mul` / `Div`: `Int`，`Float`
  - `Rem`: `Int`（`%` は `Int` のみ，0016）
  - `Concat`: `String`（0017）
  - `Eq` / `Ord`: `Int`，`Float`（数値の比較，Bool を返す）

  ユーザーが `impl Add for Int` を Emela で書くと `a + b` が自己再帰してしまう問題は，Core Prelude の
  `impl` 本体が（`a + b` ではなく）`intrinsic fn` を直接呼ぶことで回避する．孤児規則上も，これらは
  `Int` 等の所有モジュール std が提供するため，第三者による重複 impl はエラーになる．
- **暗黙の可視性（後方互換の要, MUST）**: 現状 `1 + 2` は import 無しでコンパイルできる．演算子を
  trait 化してもこれを壊さないため，**演算子 trait（`Add` 等）と上記の組み込み型 instance は，明示
  import なしに常に in-scope** とする．これは spec 0021 の **Core Prelude の暗黙 import** によって
  実現する．一方で `Show` / `to_string` を暗黙化するかは spec 0021 の選択事項とする（暗黙化しなければ
  従来どおり明示的な import を要する）．
- **ユーザー型への拡張**: ユーザー型に `impl Add for Vec2`（あるいは `impl Eq for Vec2` など）を
  与えると，その型に対して `+`（あるいは `==`）が使えるようになる．これが「加算可能な集合を `+` で
  畳む」という要求を満たす．

### 解決の限界

本仕様の境界は，**単相化により静的に discharge できるもの** に限る．次のように辞書渡し
(dictionary passing) や実行時ディスパッチを要するものは，本仕様では扱わない（コンパイルエラー，
または構文を導入しない）．

- **trait object / 存在型**（`Array<dyn Show>` のような異種コレクション，実行時 vtable が必要）:
  構文を導入しない．
- **多相再帰で特殊化が発散する場合**（`f<T>` が `f<Array<T>>` を呼ぶ等）: 有限個の特殊化に展開
  できないためエラーとする（0014 の有限特殊化前提を境界付きにも適用する）．
- **境界付きジェネリック関数の第一級化**（値として束縛・受け渡し）: 0014 が既にジェネリック関数の
  第一級化を禁止しており，境界付きでも同様に禁止を継承する．
- **戻り値位置にのみ `Self` を持つメソッド**: 引数から実装を決められないため，本仕様では宣言できない
  （trait 宣言の項）．

## Examples

trait の宣言:

```emela
trait Add {
    fn add(a: Self, b: Self) -> Self uses {}
}

trait Show {
    fn to_string(value: Self) -> String uses {}
}
```

ユーザー record への `impl`（`+` が使えるようになる）:

```emela
record Vec2 {
    x: Int
    y: Int
}

impl Add for Vec2 {
    fn add(a: Vec2, b: Vec2) -> Vec2 uses {} {
        Vec2 {
            x: a.x + b.x   -- Int の Add（組み込み instance）
            y: a.y + b.y
        }
    }
}
```

ユーザー enum への `impl`（本体で `match`）:

```emela
enum Color {
    Red
    Green
    Blue
}

impl Show for Color {
    fn to_string(c: Color) -> String uses {} {
        match c {
            Red -> "red"
            Green -> "green"
            Blue -> "blue"
        }
    }
}
```

入れ子のディスパッチ（`Vec2` の `Show` が内部で `Int` の `Show` を呼ぶ）:

```emela
impl Show for Vec2 {
    fn to_string(v: Vec2) -> String uses {} {
        "(" ++ to_string(v.x) ++ ", " ++ to_string(v.y) ++ ")"
        -- 各 to_string(v.x) は Show for Int．Self=Int は引数型から決まる
    }
}
```

パラメータ付き instance（要素が `Show` なら配列も `Show`）:

```emela
impl<T: Show> Show for Array<T> {
    fn to_string(xs: Array<T>) -> String uses {} {
        ...   -- 要素ごとに to_string(x)（Show for T）
    }
}
```

境界付きジェネリック関数（加算可能な集合を `+` で畳む）:

```emela
fn sum<T: Add>(xs: Array<T>) -> Option<T> uses {} {
    match Array.get(xs, 0) {
        None -> None
        Some(first) -> Some(fold_add(xs, 1, first))
    }
}

fn fold_add<T: Add>(xs: Array<T>, i: Int, acc: T) -> T uses {} {
    match Array.get(xs, i) {
        None -> acc
        Some(x) -> fold_add(xs, i + 1, acc + x)
    }
}

fn main() -> Option<Int> uses {} {
    sum([1, 2, 3])                      -- T = Int, Some(6)
    -- sum([Vec2 {x:1 y:2}, Vec2 {x:3 y:4}]) も同じ関数で動く（T = Vec2）
}
```

printable な集合の総称化:

```emela
import std.io.print

fn print_line<T: Show>(value: T) -> Unit uses { io } {
    print(to_string(value) ++ "\n")   -- to_string は Show，Self=T を引数から推論
}
```

既存の `to_string(5)` は `Show` 経由でそのまま解決する．さらに，`Int` と `Float` の両方に `Show`
の実装があれば，`to_string(5)`（`Int`）と `to_string(2.0)`（`Float`）は **引数型により自動で
曖昧解消** される（spec 0018 では `int.to_string` / `float.to_string` の明示修飾を要したが，trait は
これを型ディスパッチで包摂する）．

## Compilation Notes

この節は非規範的な実装上の補足である．

- **レシーバ構文の脱糖**: `recv.method(args)` の脱糖は，ドット左辺が値か否かで分岐するため，スコープが
  必要な型検査／lowering の呼び出し解決で行う（パーサでは修飾パスとの区別ができない）．参照実装は
  ドット左辺が **単純な in-scope 変数** の場合（`s.to_string()`，`v.add(w)`）を `method(recv, args)` へ
  書き換える．任意の式レシーバ（`f().to_string()` 等）は後置 `.method` のパース対応が要るため現状は
  未対応で，`to_string(f())` と書けばよい（規範上は値レシーバ一般を許すので将来拡張）．
- **単相化の拡張**: 境界の discharge と特殊化は，0014 と同じく **型検査と lowering の段階** で行う．
  推論で `T = C` が定まると，本体中の trait メソッド呼び出し（`add(acc, x)` 等）を，
  `impl Trait for C` の対応メソッドへの **直接 call** に置換し，その impl メソッドも特殊化キューに
  積む．デフォルト実装の本体，およびパラメータ付き instance（`Show for Array<T>`）は，`Self` /
  型引数を具体化して再帰的に特殊化する（`Show for Array<Int>` は `Show for Int` を要求する）．到達
  可能な特殊化は有限であること（0014）を境界付きでも前提とし，発散する多相再帰はエラーとする．
- **マングリング**: 特殊化した trait メソッドのシンボルは，`Add__Int__add` のように
  (Trait, 型, メソッド) 由来の名前にする．0014 のジェネリックマングル（`identity__Int`）や，0018 の
  パスマングル（`std__int__to_string`）とは別系統に保つ．
- **erasure（IR 不変条件, 0012）**: trait / impl / 境界は IR 化の前に完全に消去される．typed IR には
  型変数・trait・辞書は現れず，全ノードが具体型を持つという 0012 の条件を保つ．backend は具体型の
  直接 call だけを受け取り，trait 固有の処理は不要である（0017 のような専用 IR ノードも追加しない）．
- **演算子の lowering**: 演算子は desugar 後，オペランド型の instance へ discharge される．組み込み型
  （`Int` / `Float` / `String`）の instance は Core Prelude（spec 0021）の `impl` であり，その本体の
  `intrinsic fn`（spec 0021）を backend が native 命令（`i32.add`，`f64.div`，連結等）へインライン
  lower する．したがって数値演算・連結の観測可能な意味は現状（0016 / 0017）と一致し，`Int` の 0 除算
  trap（0016）などの挙動も保存される．ユーザー型の演算子は，その型の impl メソッドの call へ lower
  する．
- **`to_string` の載せ替えと後方互換**: `std.int`（`examples/stdlib/src/int.emel`）の `to_string`
  本体を `impl Show for Int { fn to_string(n: Int) -> String { ... } }` へ移設する（孤児規則上，
  `Int` の所有モジュール std に置けるので合法）．trait メソッド名は自由関数名と同じ名前空間に入る
  ため，既存の `to_string(5)` / `int.to_string(5)` はそのまま解決し続ける．移行期は，旧
  `pub fn to_string(n: Int)` を薄いラッパとして残置してもよい (MAY)．
- **非対応ケース**: trait object・発散する多相再帰・境界付き関数の第一級化・戻り値のみ `Self` の
  メソッド・具体型が見えない分離コンパイル境界は，いずれも辞書渡しを要するため本仕様では扱わない
  （Open Questions）．

## Open Questions

- **関連型 (associated types)**: `trait Iterator { type Item ... }` のような，trait に紐づく型の
  導入と単相化との両立．
- **trait 継承 / スーパークラス**: `trait Ord: Eq`（ある trait の前提として別 trait を要求する）．
- **多パラメータ trait と境界間依存**: `trait Convert<From, To>`，`<U: Convert<T>>` のように型
  パラメータが別の型パラメータに依存する境界．
- **trait object / 存在型**: `Array<dyn Show>`．辞書渡し / vtable を要するため本仕様は非対応とした．
- **明示型引数と戻り値 `Self`**: turbofish 的な明示型引数（0014 Open Questions）を導入し，
  `fn zero() -> Self` や `fn parse(s: String) -> Self` のような戻り値のみ `Self` のメソッドを
  解禁するか．（→ 0047 が戻り値のみ `Self` を **期待型解決** で部分的に解禁済み．turbofish は引き続き
  Open．）
- **effect / error 多相と subsumption**: trait メソッドの `uses` / `throws` に row 変数を許すか
  （0008 / 0014 の effect-row 多相と連動）．impl 側が trait 宣言より小さい effect（`uses` の部分集合）
  や `throws Never` を持つことを許す subsumption を導入するか（本仕様は厳密一致とした）．
- **派生 (derive)**: `derive(Show)` のように instance を機械的に自動生成する仕組み．
- **blanket / overlapping impl**: `impl<T> Show for T` や重なり合う実装を，孤児規則の大域一意性を
  壊さずに許す方法（負の制約・特定性順位など）．
- **分離コンパイル / パッケージ境界**: 呼び出し側の具体型が見えない境界で境界を discharge するための
  辞書渡しフォールバック．
- **`?` 演算子の trait 化**: 0011 が予告した error の `From` / `Into` 変換を trait で表現するか．
- **加法単位元**: 空配列の `sum` を全域にするための `Zero` / `Monoid` 相当の trait（→ 0047 で `Monoid`
  として解決）．
- **算術 trait の標準セット拡充**: `Add` 以外の算術・比較 trait を標準ライブラリとしてどこまで
  提供するか．
