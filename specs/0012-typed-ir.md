## 0012: Typed IR

Status: Draft

Emela 構文とは別に，コンパイラが保証する中間表現を定義する仕様．

### IR の条件

IR は次の条件を満たす必要がある:

```text
typed
explicit control flow
explicit allocation
explicit calls
no syntactic sugar
effect annotations preserved
source span preserved
```

### Examples

```emela
fn add1(x: Int) -> Int {
    x + 1
}
```

これが IR としての表現になった場合

```text
fn add1(x: Int) -> Int effects {} {
    %0 = const.i32 1
    %1 = add.i32 x %0
    return %1
}
```

```emela
pub fn loadUser(id: UserId) -> Option<User> uses { db } {
    ...
}
```

```text
fn loadUser(id: UserId) -> User effects { db } {
    ...
}
```

### IR 型

```text
I32
F64
Bool
Unit
StringRef
ArrayRef<T>
RecordRef<Name>
Enum<Name>
FuncRef<Signature>
```

### 明示 allocation

```text
%0 = alloc.string "alice"
%1 = alloc.array Int [1, 2, 3]
```

### 明示 call

```text
%1 = call @g
%2 = call @h()
%3 = call @f(%1, %2)
```

### source span

各 IR 命令は元のソース位置を持つ

```text
%1 = add.i32 x %0 span(file.emel:3:5-3:10)
```
