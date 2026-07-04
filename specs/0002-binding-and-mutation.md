## 0002: Bindings and Mutation

Status: Accepted

### 仕様

変数はすべて不変なものとして扱い，再代入を許可しない．(immutable)

また，`let` を予約語として，変数束縛のための命令として使用する．

```text
let x = 1
let y = x + 1
```

