## 0010: Modules, Imports, and Visibility

Status: Draft

```emela
module user

pub fn getUser(id: UserId) -> Option<User> uses { db } {
    ...
}

fn decode(raw: String) -> User throws DecodeError {
    ...
}
```

- 1ファイル 1 module
- `pub` のない定義は module private
- `pub fn` は型・effect を明示的に書く（`uses` の明示は spec 0023 により必須 (MUST)）
- import は明示的に書く

### import

```emela
import user.getUser
import time.now
```

