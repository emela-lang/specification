## 0004: Blocks and Expression Semantics

Status: Approved

- block は式である．
- `match` は式である．
- 関数本体も block expression

### Block

```emela
{
    let x = 1
    let y = 2
    x + y
}
```

この block の値は最後の式 `x + y`

最後の式が束縛であれば，`Unit` に fallback される．

```emela
{
    let x = 1
}
```

### 型規則

block の型は最後の式の型として扱われる．

```text
{ expr } : T
```

最後の式がない場合は `Unit` になる．

明示的な `return` 構文は用意しない．
