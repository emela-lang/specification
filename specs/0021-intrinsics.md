# 0021: Intrinsic Functions

Status: Draft

純粋なプリミティブ演算を，コンパイラの組み込みではなく **標準ライブラリが宣言し backend が
native 命令として供給する** 形に置き換えるための `intrinsic fn` を定義する．これにより「`Int` の
振る舞い（`+` は加算，`to_string` がある 等）」を Emela（stdlib）側に寄せ，コンパイラが保持する
プリミティブの知識を **値の表現** と **intrinsic → 命令の対応表** の 2 つに縮退させる．spec 0013
（Platform Functions）の **純粋・インライン版** であり，spec 0020（Traits）を土台とする．

## Summary

- `intrinsic fn`（本体を持たない純粋関数宣言）を導入する．stdlib がこれを宣言し，各 backend が
  **native 命令へインライン** に lower する（`extern fn` が Runtime 供給の call へ lower するのと対）．
- コンパイラは純粋プリミティブ演算の **意味を実装しない**．保持してよいのは **(a) プリミティブ値の
  表現**（`Int` = i32 等）と **(b) intrinsic → 命令の対応表** のみである．演算子の型規則や `Char`/
  `String` 変換の意味は，stdlib の `impl`（0020）と `intrinsic fn` の組み合わせに移す．
- 演算子（0020 で trait メソッドへ desugar される `+ - * / % ++ == <`）の組み込み型に対する実装は，
  stdlib の `impl ... for Int`（等）が対応する `intrinsic fn` を呼ぶ形で与える．
- **Core Prelude**：コンパイラが常に暗黙 import するコアモジュールを定め，`1 + 2` やリテラルの
  `Int` 型が明示 import なしに解決し続けるようにする（後方互換）．
- 本仕様は 0013 の設計思想（コンパイラは実体を持たず，境界が供給する）を純粋演算へ拡張し，0020 の
  「言語が提供する組み込み instance」を stdlib の Emela 実装に置き換える．

## Motivation

現状，コンパイラは `Int` / `Float` / `Char` / `String` の **意味** を各ステージにハードコードして
いる．

- 演算子 `+ - * / % ++ == <` の型規則と，各 backend の命令テーブル（`i32.add`，`f64.div`，
  `(a / b) | 0` 等）．
- `Char::from_code` / `String::from_char` / `++`（連結）の特別扱い（専用 IR ノード）．
- `to_string` は `Int` 専用の純粋 Emela 実装が stdlib にあるのみで，型ごとに振り分ける機構がない．

このため，新しい振る舞いを足すたびにコンパイラの複数ステージへ個別対応が要る．0013 は既に
「副作用の実体はコンパイラが持たず，`extern fn` で宣言して **Runtime** が供給する」という境界を
規範化した（0013: コンパイラは副作用を実装しない）．本仕様はその **純粋版** を与える：純粋な
プリミティブ演算をコンパイラから追い出し，`intrinsic fn` で宣言して **backend** が native 命令として
供給する．結果として `Int` の振る舞いは stdlib の Emela コードに集約され，コンパイラは値の表現と
命令表だけを知る小さな核になる．

## Specification

本仕様は 0013 を範として拡張し，0020（Traits）に依存する．

### intrinsic 宣言

Emela コードは，純粋なプリミティブ演算を `intrinsic fn` で宣言する．構文は `extern fn`（0013）と
同型で，**本体を持たない**．

```emela
intrinsic fn i32_add(a: Int, b: Int) -> Int uses {}
```

- `intrinsic fn` は **純粋** でなければならない．capability effect (0009) を持ってはならず，常に
  `uses {}` である (MUST)．`extern fn`（capability を伴う Runtime 呼び出し）とはこの点で異なる．
- `intrinsic fn` は **本体を書けない** (MUST NOT)．実体は backend が供給する．
- backend を名指しし，または backend ごとに分岐する構文は存在しない (MUST NOT)．したがって
  `intrinsic fn` を含む Emela ソースは backend 非依存である（0013 と同じ規律）．
- 慣習として，`intrinsic fn` は標準ライブラリが宣言し，`impl`（0020）や `pub fn`（0010）で包む．
  アプリケーションは包まれた演算子・関数を用い，`intrinsic fn` を直接書く必要はない．

### intrinsic インターフェースとカバレッジ

- 各 backend は，自身が供給できる **intrinsic の集合** を宣言する．
- executable が（`main` から推移的に）要求する intrinsic の集合が，選択された backend の供給する
  集合の **部分集合でない** 場合，コンパイルはエラーにしなければならない (MUST)．エラーは不足して
  いる intrinsic を明示すべきである (SHOULD)．（0013 の platform カバレッジ検査と同形式．）
- library（`main` を持たない compilation unit）は，要求する intrinsic の集合を metadata として
  保持してよい (MAY)．

### lowering

- `intrinsic fn` の呼び出しは，backend が **native 命令または式へインライン** に lower する．
  これは `extern fn`（platform 関数）が Runtime への **呼び出し** に lower するのと対照的である．
- intrinsic 名から native 命令への対応は各 backend が保持し，Emela ソースからは不可視である．
  同一の intrinsic 名は，どの backend でも **観測可能な意味が一致** するように lower されなければ
  ならない (MUST)（0000: ターゲット間で観測意味が変わらない）．

### コンパイラが保持するプリミティブ知識の境界

コンパイラおよび backend が組み込みで保持してよいプリミティブの知識は，次の 2 種類に限る．

1. **値の表現**：リテラルの字句・型付けと，プリミティブ値の実行時表現（`Int` = i32，`Float` =
   f64，`Char` = Unicode scalar value の整数，`Bool` = 0/1 の整数，等）．これらは backend の ABI に
   必要であり，本仕様では stdlib へ移さない．
2. **intrinsic → 命令の対応表**：上記 lowering の実体（target 依存で，ソースに出せない）．

これら以外の，**純粋プリミティブ演算の型規則や意味を，コンパイラが組み込みとして持ってはならない**
(MUST NOT)．具体的には，演算子 `+ - * / % ++ == <` の型付けと lowering は，0020 の trait 解決と
本仕様の `intrinsic fn` に委ね，コンパイラの演算子固有の型規則・命令テーブルは撤去する．

なお，条件式・ガードの被検査値が `Bool` であること（0015）などの **言語規則** は，特定プリミティブ
演算の「意味」ではないため，コンパイラが保持してよい（将来の trait 化は Open Questions）．

### Core Prelude

- コンパイラは，指定する **コアモジュール**（本仕様では仮に `std.core` と呼ぶ）を，すべての
  compilation unit に **暗黙に import** する (MUST)．
- Core Prelude は通常の Emela ソース（標準ライブラリに含まれる）であり，特別なコンパイラ組み込みでは
  ない．そこには演算子 trait（0020 の `Add` / `Sub` / … / `Eq` / `Ord` / `Concat`）と，`Int` /
  `Float` / `String` / `Char` に対するそれらの `impl`（本体が `intrinsic fn` を呼ぶ）が置かれる．
- この暗黙 import により，`1 + 2`（→ 0020 で `Add.add(1, 2)` へ desugar → `impl Add for Int` →
  `intrinsic fn`）やリテラルの `Int` 型が，明示 import なしで従来どおり解決する．
- Core Prelude が提供する名前は，明示 import と同じ名前解決（0010 / 0018）に従う．ローカル束縛や
  同一 entry の関数は，従来どおり prelude を shadow できる（0018 R6）．
- `Show` / `to_string` を Core Prelude に含めるかは選択である．含めれば総称 `to_string` も無 import
  で使え，含めなければ従来どおり明示 import を要する（→ Open Questions）．

### 演算子・変換の移設（規範の帰結）

- 演算子 `+ - * / % ++ == <` は，0020 に従い対応する trait メソッドへ desugar される．組み込み型
  （`Int` / `Float` / `String`）に対するそれらの `impl` は Core Prelude に置かれ，本体は対応する
  `intrinsic fn`（例 `i32_add`，`i32_div_s`，`i32_rem_s`，`f64_add`，`string_concat`，`i32_eq`，
  `i32_lt_s` 等）を呼ぶ．
- 0017 の `Char::from_code` / `String::from_char` は `intrinsic fn`（例 `char_from_code`，
  `string_from_char`）とし，stdlib がこれを包む．
- これらの移設後も，除算の 0 方向切り捨て・整数 0 除算 trap（0016）や，文字列連結（0017）などの
  **観測可能な意味は，対応する intrinsic の lowering によって保存** されなければならない (MUST)．

## Examples

Core Prelude（常に暗黙 import されるコアモジュール）が，intrinsic を宣言し `impl` で包む：

```emela
-- std: src/core.emel
module core

trait Add {
    fn add(a: Self, b: Self) -> Self uses {}
}

intrinsic fn i32_add(a: Int, b: Int) -> Int uses {}

impl Add for Int {
    fn add(a: Int, b: Int) -> Int uses {} {
        i32_add(a, b)
    }
}
```

`Char` / `String` の変換も intrinsic にする：

```emela
intrinsic fn char_from_code(n: Int) -> Char uses {}
intrinsic fn string_from_char(c: Char) -> String uses {}
intrinsic fn string_concat(a: String, b: String) -> String uses {}
```

アプリケーションは，従来どおり明示 import なしで `+` を書ける：

```emela
fn main() -> Int uses {} {
    1 + 2
    -- Core Prelude の暗黙 import により解決する：
    -- 1 + 2  →(0020) Add.add(1, 2)  →  impl Add for Int  →  i32_add(1, 2)  →(backend) i32.add
}
```

ユーザー型は `impl` を書くだけでよく，その本体は `Int` の intrinsic を（`+` 経由で）使う：

```emela
record Vec2 {
    x: Int
    y: Int
}

impl Add for Vec2 {
    fn add(a: Vec2, b: Vec2) -> Vec2 uses {} {
        Vec2 {
            x: a.x + b.x   -- Int の Add → i32_add（Core Prelude）
            y: a.y + b.y
        }
    }
}
```

## Compilation Notes

この節は非規範的な実装上の補足である．

- **IR**: `intrinsic fn` の呼び出しは，専用ノード `IrExpr::Intrinsic { name, args, ret }` として
  typed IR に保持する（`IrExpr::Platform` と並行）．既存の `CharFromCode` / `StringFromChar` /
  `Concat` の各ノードや，算術 `BinaryOp` は，この汎用 `Intrinsic` ノードへ統合できる（backend は
  intrinsic 名で命令テーブルを引く）．typed IR には型変数・trait は現れず，全ノードが具体型を持つ
  という 0012 の条件を保つ（0020 の erasure と両立）．
- **registry の所在**: intrinsic 名から native 命令への対応は，target 依存であり Emela ソースに
  出せないため，各 backend（Rust 実装）に保持する．コンパイラは `intrinsic fn` 宣言の **シグネチャ
  を権威** として型検査し，「その intrinsic を選択 backend が lower できるか」を検査すれば足りる．
  0013 の `platform_interface()` に相当するコンパイラ側 canonical registry を別途持つかは実装の
  裁量である（二重化して照合してもよい）．
- **既存 backend の移行**: 各 backend の `BinaryOp` 命令テーブルと，連結・`from_char` の emit を，
  intrinsic 名で引くインライン lowering に置き換える．演算子固有のコンパイラ型規則は撤去し，0020 の
  trait 解決に一本化する．
- **Core Prelude の入手**: 移設後は `1 + 2` すら stdlib（Core Prelude）に依存する．ユーザーが毎回
  パッケージ指定せずに済むよう，コンパイラは Core Prelude のソースを **埋め込み（embed）** して常に
  ロードする方式を推奨する（詳細は Open Questions）．
- **`to_string` の移設**: 既存の `Int` 専用 `to_string` は，`impl Show for Int`（0020）へ移し，桁の
  分解に用いる除算・剰余は intrinsic 経由にする．旧 `pub fn to_string(n: Int)` は当面の互換ラッパと
  して残置してよい．

## Open Questions

- Core Prelude モジュールの正式名（`std.core` / `std.prelude`）と，コンパイラへの供給方法（ソース
  埋め込み vs 常に `--package` で供給を要求する）．prelude が見つからない場合のエラー．
- コンパイラ側に intrinsic の canonical registry（0013 の `platform_interface()` 相当）を持つか，
  backend の命令表のみを真とするか．
- `Show` / `to_string` を Core Prelude に含めて無 import 化するか，従来どおり明示 import とするか．
- intrinsic ごとの定義域・trap の規範化（`Float` の剰余非対応，無効コードポイントに対する
  `char_from_code`（0017）の挙動，整数 0 除算 trap（0016）など）を，どこまで本仕様／各 backend で
  規定するか．
- 0018 R6 が特別扱いしている `Char::from_code` / `String::from_char` を，本仕様で通常の stdlib 関数へ
  移した後，当該の名前解決の特別扱いを撤去する整理．
- `Bool` と `if` 条件・比較結果などの言語規則を将来 trait 化するか（本仕様では表現・言語規則として
  コンパイラに残す）．
- 分離コンパイル（library）における要求 intrinsic 集合の metadata 保持（0013 の library 規則と同様）．
- リテラルの型（`42` が `Int`）と Core Prelude の `Int` 型の結び付け（`Int` はコンパイラ primitive の
  ままなので最小だが，将来 `Int` 自体を stdlib 型にする場合の拡張余地）．
