# lake-registry

Public package index for the Lake language.  Inspired by
[crates.io-index](https://github.com/rust-lang/crates.io-index):
this repo holds manifests only; package sources live in their own
git repositories that each version pins.

## Adding a package

1. Open an issue titled `publish: <name> <version>` describing your
   package.
2. Submit a PR adding (or updating) `packages/<first-letter>/<name>.lake`
   following the format below.  A new package needs the `package`,
   `description`, `homepage` lines plus at least one `version` line;
   subsequent versions just add another `version` line.
3. A maintainer reviews + merges.  Once merged, users can
   `dep "<name>" "<version>"` in `lake.house`.

CI (future) validates: name uniqueness, version-string not already
listed for this package, and the declared `git`+`rev` actually clones.

## Layout

```
packages/
  <first-letter>/<package-name>.lake
```

Sharded by the lowercased first letter of the package name.  Names
starting with a non-letter use `_/`.

Example paths:

- `packages/m/marine.lake`
- `packages/l/lash.lake`
- `packages/_/foo-bar.lake`

## Manifest format

Lake-native, whitespace-delimited, quoted strings, one record per
line.  Mirrors `lake-house`'s `manifest.lake` parser so the same
tokeniser handles both.

```
package "marine"
description "Per-actor web framework for Lake"
homepage "https://github.com/morphqdd/marine"

version "0.1.0" git "https://github.com/morphqdd/marine.git" rev "v0.1.0"
version "0.2.0" git "https://github.com/morphqdd/marine.git" rev "v0.2.0"
```

Required:
- `package "<name>"` — must equal the filename minus `.lake`.
- `version "<v>" git "<url>" rev "<git-ref>"` — at least one.

Optional:
- `description "<short text>"`.
- `homepage "<url>"`.
- `yank "<version>" reason "<why>"` — soft-deprecate a version; the
  resolver still serves it (to honour lockfiles) but warns.

`rev` is anything `git checkout` accepts — tag / branch / commit
SHA.  Pinning to SHA gives the most reproducible builds.

## How `lake-house` resolves `dep "marine" "0.1.0"`

1. Check `~/.lake/libs/marine/0.1.0/` — if present, use.
2. Ensure `~/.lake/index/` is up to date: clone this repo on first
   use, `git pull` on subsequent invocations.
3. Read `packages/m/marine.lake`, find the matching `version` line.
4. `git clone --depth 1 --branch <rev> <git-url> ~/.lake/libs/marine/0.1.0/`.
5. Spawn `lakec` with `MARINE_PATH=~/.lake/libs/marine/0.1.0/`.

Override the registry origin per project via `registry "<url>"` in
`lake.house`, or globally via the `LAKE_REGISTRY` env var.  Default
is this repo's HTTPS URL.

## Yanking

Once a version is in `packages/<f>/<name>.lake` it stays.  To
deprecate, add `yank "0.1.0" reason "security: rt_alloc oob"`.
Hard-removing breaks downstream lock files.
