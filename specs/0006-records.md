## 0006: Records

Status: Draft

名前付きフィールドを持つ record の仕様．

```emela
record User {
    id: Int
    name: String
}
```

```emela
let user = User {
    id: 1
    name: "alice"
}

user.name // "alice"
```
