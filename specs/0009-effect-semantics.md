## 0009: Effect Semantics

Status: Draft

effect の推論，明示，伝播，検査を定義する仕様．

### pure

```text
uses {}
```

純粋関数は effectful 関数を直接呼ぶことはできない．以下のコードはコンパイルエラーになるべきである．

```emela
fn pureFn() -> Int uses {} {
    read(path)
}
```

### Effect の伝播

```emela
fn load() -> Text uses { fs } {
    read("config.txt")
}
```

`read` が `{ fs }` を持つため， `load` もそれを持っている．

### Runtime によって規定される Effect / 組み込み Effect

Runtime が解決するべき(振る舞いが与えられるべき) Effect は以下のとおりである．

これを組み込み Effect と定義づける．Runtime はそのすべてを解決する必要はない．

組み込み Effect を Runtime が解決できない場合は，明示的にコンパイルエラーを起こすべきである．

```text
io
fs
clock
random
log
net
host
```

組み込み Effect を発生させる platform 関数と，それを Runtime（backend）が供給する境界・契約は 0013 で定義する．

### throws と effect

error は `uses` の effect row では追跡しない．error は関数型の `throws E` 節という独立した channel で追跡する (0008, 0011)．

したがって，`throw` や throwing な呼び出しの `?` は capability effect を発生させない．`throws` の有無は `uses` の内容に影響しない．

```emela
fn read(path: Path) -> String throws IoError uses { fs }
```

この関数は capability として `fs` を要求し (`uses { fs }`)，error channel として `IoError` を送出しうる (`throws IoError`)．両者は別の channel である．

