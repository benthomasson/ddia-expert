# File: .gitignore

**Date:** 2026-05-29
**Time:** 11:40

## `.gitignore` — Python Build Artifact Exclusions

### Purpose

This file tells Git which files and directories to exclude from version control. In this all-Python repository of ~37 DDIA reference implementations, it keeps three categories of generated artifacts out of commits: bytecode caches, compiled modules, and test runner caches.

### Key Components

| Pattern | What it matches | Why it's excluded |
|---------|----------------|-------------------|
| `__pycache__/` | CPython bytecode cache directories | Generated per-machine when any `.py` file is imported; contents are non-portable across Python versions |
| `*.pyc` | Compiled Python bytecode files | Redundant with `__pycache__/` on Python 3, but catches `.pyc` files placed outside `__pycache__/` (legacy Python 2 layout or manual moves) |
| `.pytest_cache/` | Pytest's inter-run cache directory | Stores last-failed info and cache plugin data; machine-local, not useful to other developers |

### Patterns

**Minimal-surface gitignore.** This file contains only what the project actually produces — no speculative entries for IDEs, virtual environments, or OS metadata. That's appropriate for a teaching/reference repo where each module is a standalone `.py` file with no build system, packaging, or deployment tooling.

### Dependencies

Every module in the repo produces `__pycache__/` on import and `.pytest_cache/` on test runs. This file affects all 37+ subdirectories uniformly — the patterns use no path prefixes, so they match at any depth in the tree.

### Flow

Git consults `.gitignore` during `git add` and `git status`. A file matching any pattern here is treated as untracked-and-ignored: it won't appear in `git status` output and `git add .` will skip it. Already-tracked files are unaffected — but nothing in this repo was ever tracked that matches these patterns.

### Invariants

- The `__pycache__/` trailing slash means only directories named `__pycache__` are ignored, not files. This is correct — CPython always creates a directory.
- `.pytest_cache/` similarly matches only the directory.
- `*.pyc` is a glob, matching any file ending in `.pyc` at any path depth.

### Error Handling

Not applicable. `.gitignore` is declarative; malformed patterns are silently treated as literals (they just won't match anything useful). These three patterns are standard and well-formed.

---

### What's notably absent

There's no entry for `*.db` (SQLite files like the ones some modules may create during tests), `*.log`, `.env`, `venv/`, or IDE directories (`.vscode/`, `.idea/`). The B-tree module writes to disk files during tests, and the WAL module creates log files — these are presumably cleaned up by test teardown or are short-lived enough that developers handle them manually.

## Topics to Explore

- [file] `b-tree-storage-engine/btree.py` — Creates on-disk page files that persist after tests; worth checking if test cleanup handles these or if `.gitignore` should cover them
- [general] `test-artifact-cleanup` — Whether test suites across all 37 modules consistently clean up disk artifacts (WAL files, B-tree pages, SSTable segments)
- [file] `hash-index-storage/bitcask.py` — Another module that writes data files to disk; same question about artifact lifecycle
- [general] `global-vs-nested-gitignore` — Whether any subdirectory needs its own `.gitignore` for module-specific artifacts

## Beliefs

- `gitignore-covers-python-only` — The `.gitignore` exclusively targets Python runtime and test artifacts; no IDE, OS, or packaging patterns are present
- `gitignore-applies-repo-wide` — All three patterns are unprefixed, so they match in every subdirectory of the repository
- `no-disk-artifact-patterns` — On-disk files created by storage-engine modules (B-tree pages, WAL segments, SSTable files) are not covered by `.gitignore`
- `pyc-pattern-is-redundant-on-py3` — The `*.pyc` glob is a safety net; Python 3 places all `.pyc` files inside `__pycache__/`, which is already ignored by the first pattern

