## 0010: Modules, Imports, and Visibility

Status: Draft

```emela
module user

pub fn getUser(id: UserId) -> Option<User> uses { db } {
    ...
}

fn decode(raw: String) -> Result<User, DecodeError> {
    ...
}
```

- 1ファイル 1 module
- `pub` のない定義は module private
- `pub fn` は 型・effect を明示的に書くことを望む(lint で警告)
- import は明示的に書く

### import

```emela
import user.getUser
import time.now
```

