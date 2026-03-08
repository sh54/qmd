---
name: rebase-upstream
description: "Use when rebasing sh54 branch onto upstream/main, syncing with tobi/qmd, or evaluating which local fixes are still needed after upstream changes"
---

# Rebase onto Upstream

## Overview

This is a fork of `tobi/qmd`. We maintain local patches (path encoding
fix, Nix packaging) on top of upstream. Periodically we rebase onto
`upstream/main` to pick up new features while keeping our patches clean.

## Remotes

- `origin` — `git@github.com:sh54/qmd.git` (our fork)
- `upstream` — `git@github.com:tobi/qmd.git` (Tobi's repo)

## Our Local Patches (as of 2026-03-08)

Two categories of commits that upstream doesn't have:

**Path fixes (PR candidates):**
- Replace lossy `handelize` with URL encoding (`encodeQmdPath`)
- Add `path` and `paths` commands to resolve qmd:// to filesystem paths

**Nix packaging (local only):**
- bun2nix for reproducible builds
- autoPatchelfHook for node-llama-cpp on NixOS
- glibc LD_LIBRARY_PATH for NixOS
- CUDA/Vulkan/musl dep ignoring in autoPatchelfHook
- XDG_CACHE_HOME for model cache

## Rebase Process

### 1. Identify unique commits

```bash
git fetch upstream
git log --oneline sh54 ^upstream/main
```

This shows ONLY commits on sh54 that aren't in upstream. PRs that
were accepted upstream will NOT appear here — git handles this
automatically since the commit hashes match.

### 2. Evaluate if fixes are still needed

For each unique commit, check if upstream addressed the same issue
differently:

```bash
# Search upstream for related changes
git log --oneline upstream/main --grep="handelize"
git grep "handelize" upstream/main -- '*.ts'

# Check upstream branches that might be related
git branch -r | grep upstream
```

**Key question for path fixes:** Does upstream still use lossy
`handelize`? If yes, our `encodeQmdPath` fix is still needed.

### 3. Rebase

```bash
git rebase --onto upstream/main upstream/main sh54
```

This replays our unique commits on top of upstream/main.

### 4. Common conflict areas

**`src/qmd.ts` imports:** Upstream adds new exports from store.ts
frequently. Resolution: keep upstream's new imports, but swap
`handelize` for `encodeQmdPath`.

**`src/qmd.ts` functions:** Upstream may remove/move functions
(e.g., `contextCheck` was folded into `status` command). Check
if removed code was moved elsewhere before keeping our version.

**`flake.nix` nativeBuildInputs:** Upstream adds build deps (python3,
cctools). Our commits add bun2nix and autoPatchelfHook. Resolution:
combine both — keep upstream's new deps AND our Nix-specific ones.

**`showHelp()` text:** Upstream rewrites help text. Our commit adds
`path`/`paths` lines. Resolution: take upstream's new help, add our
two lines.

### 5. Regenerate bun.nix

Upstream dep changes mean the old `bun.nix` is stale. Always regenerate:

```bash
ndev bun2nix
git add bun.nix bun.lock
```

Without this, `nix build` will hang during `bunNodeModulesInstallPhase`
because bun can't find new packages in the pre-built cache. See the
`local-dev` skill for details on the EACCES hang.

### 6. Build and test

```bash
# Regenerate bun.nix first (see step 5)
nix build .# -L
```

### 7. Verify after rebase

**Source checks:**

```bash
# Should show only our local patches
git log --oneline sh54 ^upstream/main

# Check no handelize references leaked back
grep -n "handelize" src/qmd.ts src/store.ts

# Check encodeQmdPath is properly imported and used
grep -n "encodeQmdPath" src/qmd.ts
```

**Smoke tests against the real index:**

IMPORTANT: The devshell sets `INDEX_PATH` and `QMD_CONFIG_DIR` to
test values. Clear them to test against the real DB:

```bash
# Version and help
./result/bin/qmd --version
./result/bin/qmd --help | grep "path"

# Search each collection (clear devshell env overrides)
INDEX_PATH="" QMD_CONFIG_DIR="" ./result/bin/qmd search "nix" -c roam -n 3 --files
INDEX_PATH="" QMD_CONFIG_DIR="" ./result/bin/qmd search "clojure" -c claude -n 3 --files
INDEX_PATH="" QMD_CONFIG_DIR="" ./result/bin/qmd search "journal" -c journal -n 3 --files

# Path fix: verify distinct paths not collapsed (the qmd-lossy test collection)
./result/bin/qmd search "lossy" -c qmd-lossy -n 3 --files
# Should show my_file.md, my-file.md, "my file.md" as SEPARATE entries

# Path resolution
INDEX_PATH="" QMD_CONFIG_DIR="" ./result/bin/qmd path "qmd://roam/<any-file>.org"
```

**What to look for:**
- `--files` output shows `qmd://` paths with original characters
  (underscores, spaces) NOT normalized to hyphens
- `path` command resolves to real filesystem paths
- No "Collection not found" errors on real collections

## Checking merged PRs

```bash
gh pr list --repo tobi/qmd --author sh54 --state merged --json number,title
```
