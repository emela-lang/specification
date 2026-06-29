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

