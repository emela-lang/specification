## 0042: Option as a Core-Prelude Enum

Status: Draft

`Option` を組み込み型から Core Prelude の通常の generic enum へ降ろし，lang-item（0041）の役割 `option` としてコンパイラに束縛する（2026-07-19 決定）．

`Option<T>` は現在コンパイラに焼き込まれた専用の直和型であり，純 Emela の generic enum である `List`（0029）や他のユーザー定義データ型（0028）と非対称である（`List` / `Result` は組み込み型ではないのに `Option` だけが組み込み）．本仕様は `Option` を std.core（0038）に `enum Option<T> { Some(T), None }` として定義し，属性 `@lang("option")`（0041）でこの enum を lang-item の役割 `option` へ束縛する．`Option` / `Some` / `None` / `match` のソース表面は不変であり（import 不要で従来どおり書ける），本仕様は観測可能な意味を変えない後方互換な内部再編である．`?` を `Option` に用いない規則（0011）は不変である．

### Summary

- `Option` を組み込みの専用型から std.core の generic enum `enum Option<T> { Some(T), None }` へ降ろす．型検査・match（0005）・単相化（0014）・IR は generic enum 機構（0028）の一般経路で扱う．
- コンパイラは `@lang("option")`（0041）でこの enum を役割 `option` へ束縛する．本仕様は役割 `option` の形状要件と特別扱いを定める（0041 L1）．
- `option` 役割の形状は「型パラメータ 1 個・値を包む variant（arity 1）1 個・空の variant（arity 0）1 個」である．present（包む）／absent（空）の役割は **arity で識別**し，enum 名・variant 名には依存しない（0041 L3）．
- lang-item の variant は，宣言された名前（`Some`/`None`）でベア（修飾なし）に構築・照合できる．これは一般の enum 構築（0018 の修飾）に対する lang-item の特権であり，従来の `Option` の表面を保つ．
- `?` は `Option` に対して定義しない（0011「Option には `?` を使わない」を維持する）．不在の伝播は `match` またはコンビネータ（`std.option`，0031）で行う．
- Core Prelude（0038）が `Option` を暗黙 import する．ユーザーソースが `Option` を再定義する，または `@lang("option")` を複数の宣言に付ける（0041 L2）のはエラーである．

### Motivation

1. **非対称の解消**．`List`（0029）は純 Emela の generic enum，`Result` は型として存在せず `throws`（0011）で表現するのに，`Option` だけがコンパイラの専用型（内部表現・パーサ・型検査・脱糖・LSP の各層に特別扱いが散在）である．generic enum（0028）と match（0005）は既に汎用で，`Option` を専用型に留める技術的理由はない．
2. **核を小さく（0000）**．0021 が算術・比較の意味を stdlib の intrinsic へ出したのと同じ方向で，`Option` の直和型としての定義と variant 構築・照合を stdlib（std.core）へ出す．コンパイラに残すのは lang-item の役割束縛（0041）1 点のみになる．
3. **lang-item の最初の利用者**．`Option` の非組み込み化は，0041 の役割束縛機構を実体化する最初の需要である．同じ機構は将来 `Result` 等にも再利用できる（0041 Open Question）．

### Specification

以下，キーワードは RFC 2119 に従う．本仕様は lang-item の役割 `option`（0041 L1）を定める．

#### Option lang-item（O 規則）

- **O1（形状）**: `@lang("option")`（0041）を付けた enum（以下 Option 型）は，型パラメータをちょうど 1 個持ち（MUST），variant をちょうど 2 個持ち（MUST），その一方は単一フィールド（arity 1）を持つ **present** variant，他方はフィールドを持たない（arity 0）**absent** variant でなければならない（MUST）．present variant のフィールド型は当該型パラメータでなければならない（MUST）．present / absent の識別は arity による．これらに反する形状はエラーである（0041 L2）．
- **O2（定義位置）**: Option 型は embedded core（std.core，0038）に定義され，Core Prelude として全コンパイル単位へ暗黙 import される（MUST）．標準の定義は次のとおりである：

  ```emela
  @lang("option")
  enum Option<T> {
      Some(T)
      None
  }
  ```

  present variant の名前は `Some`，absent variant の名前は `None` である．
- **O3（ベア構築・照合）**: Option 型の present / absent variant は，宣言された名前（`Some` / `None`）で修飾なしに構築・照合できる（MAY）．すなわち式 `None` は absent 値，`Some(e)` は `e` を包む present 値であり，パターン `None` / `Some(p)` は対応する variant に照合する．これは一般 enum の変種構築・修飾（0018）に対する lang-item の特権である．
- **O4（`?` は不可）**: `?`（0011）は Option 型に対して定義しない（MUST NOT）．`Option<T>` の値に `?` を適用するのはエラーである．不在の伝播は `match`（0005）またはコンビネータ（`std.option`，0031）で行い，不在を error channel へ橋渡しするには `Option.ok_or`（0031）を用いる（0011 No Implicit Conversion）．
- **O5（再定義の禁止）**: ユーザーソース・パッケージが，予約された `Option` という名前で型を定義することはできない（MUST NOT，0038 の予約名規則の一般則と同旨）．Option の意味は O2 の 1 定義のみが与える．
- **O6（後方互換）**: 本仕様の適用前後で，`Option` / `Some` / `None` / `match` を用いるプログラムの型付け・実行時の観測可能な振る舞い・IR の意味は不変でなければならない（MUST）．降格は内部表現の再編であり，表面言語の変更ではない．

### Examples

Core Prelude の定義（O2）——利用側は従来どおり import 不要でベアに書ける：

```emela
fn first_even(xs: Array<Int>) -> Option<Int> uses {} {
    if Array::length(xs) == 0 {
        None
    } else {
        let head = Array::get(xs, 0)
        if head % 2 == 0 { Some(head) } else { None }
    }
}
```

match で扱う（O3，0005）——`?` は使わない（O4，0011）：

```emela
fn label(o: Option<Int>) -> String uses {} {
    match o {
        Some(n) -> "some: " ++ to_string(n)
        None -> "none"
    }
}
```

`?` を Option に適用するのはエラー（O4）：

```emela
fn bad(o: Option<Int>) -> Option<Int> uses {} {
    let n = o?     -- error: `?` is not defined for `Option<_>`; use `match` or `std.option` (spec 0011/0042)
    Some(n)
}
```

`Option` の再定義はエラー（O5）：

```emela
enum Option<T> { Some(T), None }
-- error: `Option` is a reserved core type (spec 0042); it is provided by std.core
```

### Compilation Notes

この節は非規範的な実装上の補足である．

- コンパイラは組み込みの専用型 `Type::Option` を廃し，Option 型を一般の `Type::Enum("Option", …)` として扱う．型検査・unify・置換・単相化名（mangling）・IR ダンプ表示・LSP 型表示は generic enum の既存経路が同一の結果を出すため，`Option` 専用の分岐は削除できる．
- 役割 `option` の lang-item テーブル（0041 Compilation Notes）は，present（arity 1）/ absent（arity 0）variant の名前とタグを記録し O1 の形状を検証する．ベア `Some` / `None`（O3）の名前解決と match の網羅性検査はこのテーブルを参照する（名前 `"Option"` を直接照合しない，0041 L3）．
- 実装は，0011 が禁じているにもかかわらず存在していた `?`-on-Option（`Option<T>` に対する `?`）を削除し，0011 への準拠を回復する．これは本降格に伴う conformance 修正であり，別の新規規則ではない（O4 が現状の規範を再確認する）．
- backend の enum 表現（タグ + フィールド）は不変であり，Option もその一般表現に載る．`?`-on-Option の削除により，従来 backend が持っていた不在値の sentinel と境界処理（None 伝播のための）は不要になる．
- present / absent を arity で識別するため，実装は variant 名 `Some` / `None` をハードコードしない．標準定義（O2）では宣言順から present=tag 0 / absent=tag 1 となり，従来の内部タグと一致する．

### 他仕様との関係

- **0041（Lang Items）**: `@lang("option")` で役割 `option` を束縛する．役割束縛の枠（L 規則）は 0041 が，役割 `option` の形状・特別扱い（O 規則）は本仕様が定める．
- **0039（Attributes）**: `@lang` は 0039 の引数付き属性であり，`option` はその引数（役割名）である．
- **0038（Core Package Embedding）**: `Option` の定義は embedded core（std.core）に置かれ，暗黙 import される．予約名規則により `Option` の再定義は禁止される（O5）．
- **0028（Generic Data Types）/ 0005（Enum and Match）/ 0014（Generic Functions）**: 降格後の `Option` はこれらの一般機構だけで型検査・照合・単相化される．0028 の Open Question「組み込み `Option` / `Array` を本機構上の宣言として再定義するか」に対し，`Option` については「する」と答える（`Array` は組み込みプリミティブ型として別途，0007）．
- **0011（Error Handling）**: 「Option には `?` を使わない」を維持し（O4），実装の非準拠な `?`-on-Option を削除して 0011 準拠を回復する．`throws` と `Option` の間に暗黙変換はなく，橋渡しは `Option.ok_or`（0031）である点も不変．
- **0031（Stdlib Option Combinators）**: `std.option` のコンビネータは降格後の `Option` enum に対してそのまま動作する（変更不要）．本仕様は型 `Option` の所在を定め，0031 はそのコンビネータを定める（役割分担）．
- **0001（Core Values and Types）/ 0012（Typed IR）/ 0033（Language Server）**: `Option` を「組み込み型」と述べる記述は「Core Prelude が提供する型」に読み替える．IR / LSP の `Option` 専用処理は generic enum の一般処理へ統合される．

### Open Questions

- lang-item の variant にベア構築の特権を与える（O3）代わりに，一般の enum 変種をベアで構築・照合できるようにする言語変更（0005/0018 の拡張）を行うか．行えば O3 の特権は不要になるが，曖昧性解決の設計が別途要る．
- `Result` を将来 stdlib の値型として導入し，`@lang("result")`（0041）で降ろすか（0011 は現状「`Result` は組み込みではない」とし `throws` で表現）．役割 `result` を足す形になる．
