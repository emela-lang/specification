# 0012: Toolchain and Package Resolution

Status: Draft

## Summary

このドラフトは、Emela toolchain binary と package 解決の責務境界を定義します。

Emela は compiler、backend 実行、package resolver/fetcher/cache を同一 toolchain binary
に同居させてよいです。ただし、compiler の意味論は package の取得や version solving に
依存してはなりません。compiler は、解決済み source package directory の集合を受け取り、
import 解決に使うだけです。

## Motivation

標準ライブラリを外部 repository として発展させるには、compiler が source package を
読み込める必要があります。一方で、標準ライブラリと package resolver が利用不能でも、
toolchain binary 単体で最低限のプログラムを check/build できる状態は保ちたいです。

このため、canonical な stdlib は外部 package としつつ、toolchain binary は compiler と
互換な embedded stdlib fallback を持てます。

## Specification

### Toolchain Binary

実装は、単一の toolchain binary `emela` を提供してよいです。この binary は少なくとも
次の責務を持てます。

- `check`, `build`, `compile` などの compiler entrypoint。
- project manifest と lock file の読み込み。
- package dependency の解決。
- package の取得と cache への展開。
- backend profile と bundled runtime asset の選択。

これらは同一 binary に入ってよいですが、compiler core は package 取得、network access、
version solving、lock file 書き換えを行ってはなりません。

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

`version` が存在する場合、lock file と dependency resolution で使われます。compiler core
は `version` を解釈してはなりません。

### Project Manifest

project root は `emela.json` を持てます。初期 project manifest は package identity と
dependency set を持ちます。

```json
{
  "package": {
    "name": "app",
    "version": "0.1.0"
  },
  "dependencies": {
    "std": {
      "git": "https://github.com/emela-lang/std",
      "rev": "0123456789abcdef"
    }
  }
}
```

初期 toolchain は dependency source として local path と git revision を support して
よいです。registry、semver range solving、feature flag はこの仕様では定義しません。

### Lock File

toolchain は解決済み dependency graph を `emela.lock` に固定できます。lock file は、
各 package の name、version、source identity、resolved revision、cache path または
content identity を記録します。

```json
{
  "packages": [
    {
      "name": "std",
      "version": "0.1.0",
      "source": {
        "kind": "git",
        "url": "https://github.com/emela-lang/std",
        "rev": "0123456789abcdef"
      }
    }
  ]
}
```

`emela.lock` が存在する場合、build/check は lock file に従って package set を構築する
SHOULD です。manifest と lock file が矛盾する場合、toolchain は reject するか、明示的な
update/fetch command を要求する SHOULD です。

### Package Cache

toolchain は package を user cache directory へ展開してよいです。cache layout は実装依存
です。ただし、compiler core へ渡す時点では、各 package は通常の source package directory
として見える必要があります。

### Standard Library

canonical stdlib は package `std` として外部 repository に置けます。stdlib repository は
`emela-package.json` を持ちます。

```json
{
  "name": "std",
  "version": "0.1.0",
  "source": "std"
}
```

toolchain binary は embedded stdlib fallback を持ってよいです。package `std` の解決順位は
次の通りです。

1. project lock file または explicit package set に含まれる `std` package。
2. package cache から解決された `std` package。
3. toolchain binary に embedded された stdlib fallback。

embedded stdlib fallback は、toolchain binary と互換な最低限の stdlib snapshot です。
canonical stdlib repository の代替ではありません。

### Compiler Interface

低レベル compiler CLI は、解決済み source package を `--package DIR` で受け取ってよいです。
`DIR` は `emela-package.json` を持たなければなりません。

`--package` は package manager の代替ではなく、toolchain が構築した package set を compiler
へ渡すための escape hatch です。

標準ライブラリ専用の `--stdlib` option は定義しません。stdlib を外部から与える場合も、
package `std` として `--package` または project dependency を使います。

### Commands

初期 toolchain は次の command を提供できます。

```text
emela check [INPUT]
emela build [INPUT]
emela package init
emela package add NAME SOURCE
emela package fetch
emela package update
```

command 名と exact CLI syntax は実装依存です。ただし、package 解決 command と compiler
entrypoint は同一 binary に同居してよいです。

## Examples

外部 stdlib repository を project dependency として使う例:

```json
{
  "dependencies": {
    "std": {
      "git": "https://github.com/emela-lang/std",
      "rev": "0123456789abcdef"
    }
  }
}
```

低レベル compiler entrypoint で解決済み stdlib package を渡す例:

```sh
emela compile --package ../std examples/std-print.emel
```

## Compilation Notes

package manager は package graph を解決した後、compiler core に package set を渡します。
compiler core は import path の先頭 package name を package set と embedded stdlib fallback
から解決します。

この分離により、将来 package manager を Emela 自身で実装しても、compiler core の import
意味論を変更せずに差し替えられます。

## Open Questions

- registry protocol を定義するか。
- semver range と feature flag を導入するか。
- lock file を JSON に固定するか、別形式にするか。
- package metadata から型情報だけを読む compiled metadata format を導入するか。
