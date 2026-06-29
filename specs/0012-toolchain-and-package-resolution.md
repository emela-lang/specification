# 0012: Toolchain and Package Resolution

Status: Draft

## Summary

このドラフトは、Emela toolchain binary と package 解決の責務境界を定義します。

Emela は compiler、backend 実行、package resolver/fetcher/cache を同一 toolchain binary
`emela` に同居させます。ただし、compiler の意味論は package の取得や version solving に
依存してはなりません。compiler は、解決済み source package directory の集合を受け取り、
import 解決に使うだけです。

## Motivation

標準ライブラリを外部 repository として発展させるには、compiler が source package を
読み込める必要があります。標準ライブラリは通常の package dependency として扱い、
project manifest または explicit package set から解決します。

## Specification

### Toolchain Binary

実装は、単一の toolchain binary `emela` を提供します。この binary は少なくとも
次の責務を持ちます。

- `check`, `build` の compiler entrypoint。
- project manifest の読み込み。
- package dependency の解決。
- package の取得、manifest への追加、cache への展開。
- backend profile と bundled runtime asset の選択。

これらは同一 binary に入ってよいですが、compiler core は package 取得、network access、
version solving、package cache 書き換えを行ってはなりません。

compiler core が受け取る package 情報は、解決済み source package directory の集合です。

```text
PackageSet =
  PackageSource*

PackageSource =
  package name
  source root directory
```

### Source Package Manifest

source package directory は root に `emela-package.json` を持ちます。最小 manifest は
[0008: Imports](0008-imports.md) で定義する `name` と `source` を持ちます。

toolchain は追加で `version` を読んでよいです。

```json
{
  "name": "std",
  "version": "0.1.0",
  "source": "std"
}
```

`version` が存在する場合、toolchain metadata として使えます。compiler core は `version`
を解釈してはなりません。

### Project Manifest

project root は `emela.json` を持てます。初期 project manifest は package identity と
dependency set を持ちます。v1 の dependency は git URL と revision を固定します。

```json
{
  "package": {
    "name": "app",
    "version": "0.1.0"
  },
  "dependencies": {
    "std": {
      "git": "https://github.com/emela-lang/stdlib.git",
      "rev": "0123456789abcdef"
    },
    "math": {
      "git": "https://github.com/example/emela-math.git",
      "rev": "abcdef0123456789"
    }
  }
}
```

v1 toolchain は dependency source として git revision を support します。registry、
semver range solving、feature flag、branch/tag tracking はこの仕様では定義しません。

### Lock File

v1 は lock file を定義しません。再現性は `emela.json` の git URL と `rev` によって確保
します。`emela.lock` の形式、生成、更新、検証は後続仕様で定義します。

### Package Cache

toolchain は package を user cache directory へ展開します。cache root は
`$EMELA_HOME/cache`、`EMELA_HOME` が未設定なら `$HOME/.emela/cache` です。git dependency
の cache layout は次の通りです。

```text
cache/git/<sanitized-git-url>/<rev>/
```

compiler core へ渡す時点では、各 package は通常の source package directory として見える
必要があります。

### Standard Library

canonical stdlib は package `std` として外部 repository に置きます。stdlib repository は
`emela-package.json` を持ちます。

```json
{
  "name": "std",
  "version": "0.1.0",
  "source": "std"
}
```

package `std` は次のどちらか一方で解決されます。

1. explicit package set に含まれる `std` package。
2. project manifest dependency に含まれる `std` package。

同じ compile command で両方に `std` package が含まれる場合、toolchain は duplicate package
として reject します。

embedded stdlib fallback は定義しません。`std.*` import を使う project は、package
`std` を `--package` または project dependency で明示しなければなりません。

### Compiler Interface

低レベル compiler CLI は、解決済み source package を `--package DIR` で受け取ってよいです。
`DIR` は `emela-package.json` を持たなければなりません。

`--package` は package manager の代替ではなく、toolchain が構築した package set を compiler
へ渡すための escape hatch です。

標準ライブラリ専用の `--stdlib` option は定義しません。stdlib を外部から与える場合も、
package `std` として `--package` または project dependency を使います。

### Commands

v1 toolchain は次の command を提供します。

```text
emela check [--backend PROFILE] [--package DIR]... INPUT.emel
emela build --backend PROFILE [--artifact PATH | --output PATH] [--package DIR]... INPUT.emel
emela package fetch
emela package add NAME --git URL --rev REV
```

`--package DIR` は低レベル escape hatch です。project manifest dependency と同じ package
name を `--package` で同時に渡した場合、toolchain は reject します。

`emela package fetch` は project root の `emela.json` を読み、cache miss の dependency を
`git clone` し、指定された `rev` を checkout します。取得後、dependency key と
`emela-package.json` の `name` が一致しなければ reject します。

`emela package add NAME --git URL --rev REV` は project root の `emela.json` を読み、
dependency `NAME` を追加し、同じ dependency を cache に取得します。`NAME`、`URL`、`REV`
は空であってはなりません。既存 dependency と同じ `NAME` は reject します。取得後、
dependency key と `emela-package.json` の `name` が一致しなければ reject します。

`emela check` と `emela build` は project manifest dependency を cache 上の source package
directory として解決しますが、missing dependency を fetch してはなりません。cache に
存在しない dependency がある場合、toolchain は `emela package fetch` を促す diagnostic を
出して reject します。

project root は input file から親方向に最初に見つかる `emela.json` の directory です。
`emela.json` がない場合、build/check は manifest dependency なしで動作します。

## Examples

外部 stdlib repository を project dependency として使う例:

```sh
emela package add std --git https://github.com/emela-lang/stdlib.git --rev 0123456789abcdef
```

```json
{
  "dependencies": {
    "std": {
      "git": "https://github.com/emela-lang/stdlib.git",
      "rev": "0123456789abcdef"
    }
  }
}
```

低レベル compiler entrypoint で解決済み stdlib package を渡す例:

```sh
emela check --package ../std examples/std-print.emel
```

## Compilation Notes

package manager は package set を解決した後、compiler core に package set を渡します。
compiler core は import path の先頭 package name を package set から解決します。

この分離により、将来 package manager を Emela 自身で実装しても、compiler core の import
意味論を変更せずに差し替えられます。

## Open Questions

- registry protocol を定義するか。
- semver range と feature flag を導入するか。
- lock file を導入するか。導入する場合、JSON に固定するか、別形式にするか。
- package metadata から型情報だけを読む compiled metadata format を導入するか。
