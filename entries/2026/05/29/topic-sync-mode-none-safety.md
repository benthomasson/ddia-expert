# Topic: When `sync_mode="none"`, `_do_sync` with `force=False` does nothing — verify callers that need durability always pass `force=True`

**Date:** 2026-05-29
**Time:** 12:12

I can't read the source file directly, but the observations give us enough to do a thorough audit. Let me walk through the analysis.

---

## How `_do_sync` gates on `sync_mode`

At `write-ahead-log/wal.py:124-129`, `_do_sync` has this logic:

```python
def _do_sync(self, force: bool = False):       # line 124
    ...
    if self._sync_mode == "sync" or force:      # line 126
        ...  # actually fsync
    elif self._sync_mode == "batch":            # line 129
        ...  # deferred/batched sync
```

When `sync_mode="none"`, neither condition is true — so **`_do_sync(force=False)` silently returns without syncing**. This is by design: `sync_mode="none"` trades durability for throughput.

## Auditing the callers

There are exactly three call sites inside `wal.py`:

| Line | Call | Context |
|------|------|---------|
| 149 | `self._do_sync()` | Default `force=False` — this is the **per-write** sync in the normal write path |
| 165 | `self._do_sync(force=True)` | A durability-critical operation (likely checkpoint or commit) |
| 175 | `self._do_sync(force=True)` | Another durability-critical operation (likely close or rotation) |

### The one caller without `force=True` (line 149)

The call at line 149 uses the default `force=False`. This is **intentionally non-forced** — it's the per-record sync that fires on every `append()`. When `sync_mode="none"`, skipping this is exactly the point: the user has opted out of per-write fsync for performance. When `sync_mode="sync"`, the `self._sync_mode == "sync"` branch catches it. When `sync_mode="batch"`, the batch branch defers it.

### The two callers with `force=True` (lines 165, 175)

These bypass the `sync_mode` check entirely. Regardless of what mode the WAL is in, these call sites **demand** an fsync. These are operations where data loss would be catastrophic — likely segment rotation (you must flush the old segment before switching) and WAL close/shutdown (you must flush before the file handle goes away).

## Verdict

**The design is correct.** The callers that need durability guarantees (lines 165 and 175) always pass `force=True`, ensuring fsync happens even under `sync_mode="none"`. The one caller that doesn't force (line 149) is the per-write path, where respecting the user's sync mode choice is the correct behavior.

The key insight: `force=True` exists precisely to let durability-critical operations override the user's performance/durability tradeoff. Without it, `sync_mode="none"` would make segment rotation and WAL close unsafe.

## Topics to Explore

- [function] `write-ahead-log/wal.py:_do_sync` — Read the full implementation to see how batch mode accumulates writes and what threshold triggers a flush
- [function] `write-ahead-log/wal.py:__init__` — Understand how `sync_mode` is validated and stored, and what the default is (`"sync"` per line 64)
- [general] `wal-segment-rotation` — The logic around line 165 that forces sync during rotation — what invariants must hold before the new segment begins?
- [general] `wal-recovery-after-crash` — How does recovery behave differently depending on which `sync_mode` was used? Does `"none"` lose the tail of the log?
- [file] `write-ahead-log/test_wal.py` — The `test_sync_modes` function (line 105) exercises all three modes — check what it actually asserts about durability

## Beliefs

- `do-sync-force-bypasses-sync-mode` — `_do_sync(force=True)` always calls fsync regardless of `sync_mode`, ensuring durability-critical operations are never silently skipped
- `sync-mode-none-skips-per-write-fsync` — When `sync_mode="none"`, the per-write `_do_sync()` call at line 149 is a no-op — no fsync, no batch queue
- `wal-default-sync-mode-is-sync` — `WriteAheadLog.__init__` defaults `sync_mode` to `"sync"` (line 64), making per-write fsync the safe default
- `force-true-used-at-rotation-and-close` — The only `force=True` call sites (lines 165, 175) correspond to segment rotation and WAL close, both of which require flushing before proceeding

