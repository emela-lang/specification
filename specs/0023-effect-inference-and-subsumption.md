# 0023: Effect Inference and Subsumption

Status: Draft

関数定義の `uses` 節の **推論**，注釈の意味（上界検査），row の部分集合による適合（**subsumption**），
および **row 拡張** `uses { Io, ..e }` を定義する仕様．0009（Effect Semantics）と 0022
（Effect-Row Polymorphism）を拡張し，0010 の lint を規範へ格上げする．

2026-07-23 改訂: 0022 の改訂（row 変数の `<...>` 明示宣言・sigil 廃止）に追随し，row 拡張の構文を
`..'e` から `..e` へ変更，subsumption を tail 込みの成分ごと包含へ拡張した．

## Summary

- **関数定義**で `uses` 節を省略した場合，effect row は本体から**推論**される（従来の「省略 = `uses {}`」
  を定義について廃止する）．**関数型の表記**における省略は従来どおり `uses {}` を意味する（0008 不変）．
- `pub fn` は `uses` を明示しなければならない (MUST)（0010 の lint を規範化）．
- `uses` 注釈は**上界**である: 推論された row ⊆ 宣言された row でなければならない (MUST)．過大宣言は
  合法（将来予約に使える）．
- **Subsumption**: 具体 row `F1 ⊆ F2` のとき，`(A) -> T uses F1` の関数値は `(A) -> T uses F2` が
  期待される位置に適合する (MUST)．
- **Row 拡張**: `uses { Io, ..e }` は「`Io` に加えて row 変数 `e` の内容」を表す．0022 の第一段階
  制限（具体 row か単一変数のみ）を緩和する．

## Motivation

0008 は「`uses {}` は省略可能」としか定めておらず，effect を持つ関数はすべて注釈を要した．これは
「初学者は effect を知らずに書き始められる」（0000 原則5, progressive disclosure）に反する．また row が
正確一致でしか適合しない（0003）ため，`uses { fs }` の関数値を `uses { fs, net }` を期待する高階関数へ
渡せず，effect 型が脆かった．さらに 0022 は「高階関数自身が effect を持ちつつ callback の effect も
足す」形（row 拡張）を将来に送っていた．本仕様はこの3つを閉じる．

## Specification

### 省略の意味（定義と型で異なる）

- **関数型の表記**（型注釈・引数の関数型など）で `uses` を省略した場合，その型は `uses {}`（純粋）で
  ある (MUST)（0008 不変）．型は明示的な契約であり，推論されない．
- **関数定義**（名前付き `fn`・無名関数）で `uses` 節を省略した場合，effect row は本体から推論される
  (MUST)．

### 推論

- 推論される row は，0009 / 0005 / 0015 の合成規則（呼び出し・guard・分岐の和集合）に従って本体から
  計算される**最小の row** である (MUST)．
- 再帰・相互再帰は最小不動点で解く: 各関数の row を `{}` から始め，変化がなくなるまで和集合を反復する．
  effect の全体集合は有限（0009 + 0026）なので停止する．
- effect-row 多相な関数（0022）の呼び出しは，row 変数を実引数から具体化した後の row を合成に用いる．
  呼び出し元自身が row 多相なら，合成される row に呼び出し元の row 変数が tail として現れてよい．
- 推論が row 変数を**導入する**ことはない (MUST NOT)．effect-row 多相は `<...>` への明示宣言に
  よるオプトインのみである（0022）．推論はシグネチャに書かれた row 変数を伝播するだけである．
- 無名関数も同様に推論する．`uses` を明示した無名関数は下記の上界検査に従う．

### pub 境界の明示

- `pub fn` は `uses` 節を明示しなければならない (MUST)．省略はコンパイルエラーとする（0010 の
  「lint で警告」を規範へ格上げする）．public API の effect は推論結果ではなく宣言である．
- `extern fn`（0013）と `intrinsic fn`（0021）は従来どおり常に明示である．

### 注釈は上界

`uses` 節を明示した関数定義では，推論された row を `I`，宣言された row を `D` として

- `I ⊆ D` でなければならない (MUST)．`I ⊄ D` はコンパイルエラーであり，不足している effect を列挙
  すべきである (SHOULD)．
- `D ⊋ I`（過大宣言）は合法である．将来 effect を追加する余地を API に予約する用途に使える．lint は
  設定により警告してよい (MAY)．

過大宣言された effect は manifest（0025）にも現れる: プログラムの要求集合は宣言 row ではなく，実際に
到達する platform 関数から計算される（0013）ため，過大宣言が Runtime 要求を増やすことはない．

### 派生 effect の discharge（0049）

- 本仕様の推論・和集合・上界検査・subsumption が扱う row の要素は，プリミティブ effect と**派生 effect**
  （0037/0049）の双方でありうる．これらの検査はこの **source row**（派生を含みうる）の上で従来どおり行う．
- source row から，各派生 effect をその handler の依存 row で置換して **leaf row** を得る **discharge**
  （0049 D2）は，推論の後段の変換である．manifest（0025）・0013 のカバレッジ検査・エントリ境界の全解決
  検査（0049 D6）はいずれも leaf row に対して働く．
- 未解決の派生 effect（handler が無い，0049 H6）は discharge されずに leaf row に residue として残り，
  境界で 0049 D6 のエラーになる．

### Subsumption（部分集合による適合）

具体 row `F1`, `F2` について `F1 ⊆ F2` のとき，

- 型 `(A, ...) -> T throws E uses F1` の関数値は，型 `(A, ...) -> T throws E uses F2` が期待される
  あらゆる位置（引数渡し・注釈付き束縛・戻り値・record フィールド）に適合する (MUST)．
- 関数型がネストする場合は通常の変性に従う: パラメータ位置の row は逆向き（より大きい row を受ける
  関数はより小さい row を期待する位置に適合する），結果位置は同向きに比較する．

```emela
fn twice(f: (Int) -> Int uses { fs, net }) -> Int uses { fs, net } { ... }

fn pure_inc(x: Int) -> Int { x + 1 }        -- uses {} (推論)

twice(pure_inc)    -- OK: {} ⊆ { fs, net }
```

0003 の「`uses` の異なる関数型は異なる型」は維持される．subsumption は同一性ではなく**適合**
（部分型方向の変換）である．

### Row 変数との関係（tail 込みの包含）

row 変数（0022）を含む row の包含は，具体部と tail 集合を **成分ごと** に判定する (MUST)．
row `{ C1, ..T1 }`（具体部 `C1`，tail の集合 `T1`）と `{ C2, ..T2 }` について

```text
{ C1, ..T1 } ⊆ { C2, ..T2 }   ⟺   C1 ⊆ C2  かつ  T1 ⊆ T2
```

row 変数は全称量化されている（あらゆる具体化で包含が成り立つ必要がある）ため，この規則は健全かつ
完全である: 全 tail を `{}` に具体化すれば `C1 ⊆ C2` が必要であり，`T1 ⊄ T2` なら余分な tail に
fresh な effect を与えることで反例が作れる．

- `uses e` に対する一致は 0022 の単一化に従い，最小解を取る（下記 row 拡張参照）．
- tail を通した**緩和**（bounded row variable，`e ⊆ { Fs, Net }` のような上界制約）は引き続き
  導入しない（Open Questions）．上の規則は同名 tail の共有を要求する包含であり，緩和ではない．

### Row 拡張

`uses` 節に，具体 effect と row 変数（`..e`）を並べた **拡張 row** を書ける．

```text
uses { Io, ..e }
```

- 意味は集合の和 `{ Io } ∪ e` である．`e` が `{ Io, Fs }` に具体化されれば全体は `{ Io, Fs }`
  （重複は正規化される）．
- 出現位置は 0022 と同じ（パラメータの関数型の `uses`，当該関数自身の `uses`）．パラメータの
  関数型の `uses` に書ける tail は最大 1 個，当該関数自身の `uses` には複数書いてよい（0022）．
- 単一化は**最小解**を取る: 実引数の row `(A, tail Ta)` を拡張 row `{ D, ..e }` に対応付けるとき，
  `e = (A \ D) ∪ Ta` とする (MUST)．実引数が具体 row `C`（tail なし）の特殊ケースでは従来どおり
  `e = C \ D` である．
- 0022 の「第一段階では具体 row か単一変数に限る」制限は，本仕様により緩和される．

```emela
-- callback を実行しつつ自身も log する高階関数
fn traced<T, U, e>(x: T, f: (T) -> U uses e) -> U uses { Log, ..e } {
    Log.info("call")     -- uses { Log }
    f(x)                 -- uses e
}
```

## Examples

推論（private 関数は注釈不要）:

```emela
fn load() -> String {          -- uses { fs } と推論される
    read("config.txt")         -- read: uses { fs }
}

pub fn main() -> Unit uses { io, fs } {   -- pub は明示必須
    print(load())
}
```

上界検査:

```emela
fn f() -> Unit uses { io } {
    read(path)                 -- error: 推論 { fs } ⊄ 宣言 { io }
}

fn g() -> Unit uses { io, fs } {
    print("hi")                -- OK: 推論 { io } ⊆ 宣言 { io, fs }（過大宣言は合法）
}
```

## Compilation Notes

この節は非規範的である．

- 推論は effect 検査（0009）と同じ走査で行える: 検査が「宣言に含まれるか」を見る代わりに，推論は
  「和集合に加える」だけである．
- subsumption は row の部分集合判定であり，実行時表現を持たない（0022 と同じく erase される）．
  coercion コードは生成されない．
- typed IR（0012）には具体化・推論済みの row のみが現れる．派生 effect は discharge 後 primitive leaf に
  還元され，handler ともども IR には現れない（0049 D3）．

## Open Questions

- `throws` の推論（private 関数で `throws` を省略し本体から推論する）を同様に導入するか．
- bounded row variable（`e ⊆ { Fs, Net }` のような上界付き row 変数）の導入．
- 過大宣言に対する lint の既定（警告 on/off）．
- IDE / ツールが推論された row を inlay hint として表示する規約．
- `pub fn` の `uses` も推論可能にし（progressive disclosure を pub 境界まで広げる），推論結果を
  LSP・型チェックで提示するか（Track 4）．現状は `pub fn` は明示必須（本仕様）．
