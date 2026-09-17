# Probe: checksum-signing-git-source

## Metadata

| Field                  | Value                                           |
|------------------------|-------------------------------------------------|
| Pattern name           | checksum-signing-git-source                     |
| Package manager        | Bundler (Ruby)                                  |
| PM version under test  | 4.0.21                                          |
| Schema version         | 1.2                                             |
| Categories             | registry_source, checksum_signing               |
| Generated at           | 2026-09-17T12:00:00Z                            |

## What this probe exercises

This probe targets two behavioural changes introduced in **Bundler 4.0.21**:

### 1. Git source security — `safe.bareRepository=explicit`

Bundler 4.0.21 sets the Git config option `safe.bareRepository=explicit`
when cloning or fetching git-sourced gems. This affects how the Mend
Unified Agent resolves the git-source entries in `Gemfile.lock`.

The probe includes `rack-test` pinned to a specific commit via a `git`
source block (tag `v2.1.0`, commit
`7a5b6e7e95c78d52e9ceef57f8db1d0b0f95a1c3`). The UA must follow the
`GIT` section of the lockfile and correctly identify the gem name and
version from the nested `specs:` block, treating it the same as any
other resolved gem but noting the `git` source type.

### 2. Checksum validation — empty `CHECKSUMS` entries

Bundler 4.0.21 changed handling of empty `CHECKSUMS` slots in
`Gemfile.lock`. Previously an absent checksum hash was treated one way;
now the UA must tolerate entries where the value is present but carries
no hash string (e.g. `sinatra (4.0.0)` with no `sha256=` suffix).

The lockfile in this probe contains:
- `mustermann (3.0.0)` — full sha256 checksum present
- `rack (3.1.8)` — full sha256 checksum present
- `sinatra (4.0.0)` — **empty checksum entry** (no sha256 value)

The `rack-test` git-sourced gem has no `CHECKSUMS` entry at all, which
is the standard Bundler behaviour for git sources; this is distinct
from an empty entry for a registry gem.

## Dependency graph

```
(root)
  ├── rack 3.1.8          [registry] — direct, checksum present
  ├── sinatra 4.0.0       [registry] — direct, empty checksum entry
  │   ├── mustermann 3.0.0  [registry] — transitive, checksum present
  │   └── rack 3.1.8        [registry] — transitive (shared with direct)
  └── rack-test 2.1.0     [git]      — direct, no checksum entry
      └── rack 3.1.8        [registry] — transitive (shared)
```

## Files

| File            | Purpose                                           |
|-----------------|---------------------------------------------------|
| `Gemfile`       | Project manifest; declares all three direct deps  |
| `Gemfile.lock`  | Fully resolved lockfile matching Bundler 4.0.21   |
| `.whitesource`  | Pins `bundler: 4.0.21` and `ruby: 3.3.0`         |
| `expected-tree.json` | Ground-truth dependency tree for downstream  |

## Mend config

**Bucket A** — `ruby-bundler` has no dynamic version detection in the
Mend Unified Agent. This probe ships a `.whitesource` that pins:

```json
{
  "scanSettings": {
    "configMode": "AUTO",
    "versioning": {
      "bundler": "4.0.21",
      "ruby": "3.3.0"
    }
  }
}
```

`configMode` is `"AUTO"` because no `whitesource.config` is present in
this probe root. The UA resolves via lock-file parsing
(`ruby.resolveDependencies=true` by default); no `bundle install` is
needed because the lockfile is fully resolved.

## Resolver behaviour notes

The Mend UA Ruby resolver will:

1. Detect Bundler 4.0.21 (Bundler 2.x+ priority: `Gemfile.lock`).
2. Parse the `GIT` section first to register `rack-test 2.1.0` as a
   git-sourced gem with revision
   `7a5b6e7e95c78d52e9ceef57f8db1d0b0f95a1c3`.
3. Parse the `GEM.specs` section for registry gems.
4. Parse `CHECKSUMS` — tolerate the empty entry for `sinatra (4.0.0)`.
5. Read `DEPENDENCIES` to identify the three direct gems.
6. Build the tree: `rack-test` and `sinatra` each pull in `rack` as a
   transitive; the resolver's cycle/shared-node handling collapses
   `rack` to a single node.
