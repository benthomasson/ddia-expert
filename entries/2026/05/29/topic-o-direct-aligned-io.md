# Topic: How `O_DIRECT` requires `posix_memalign` or `mmap` for buffer allocation, and why Python's `open()` can't satisfy these constraints without ctypes

**Date:** 2026-05-29
**Time:** 11:04

Now I have all the context I need. Let me write the explanation.

---

# How `O_DIRECT` Requires Aligned Buffers, and Why Python Can't Provide Them Natively

## The Core Constraint

When you open a file with `O_DIRECT`, the kernel removes itself from the data path. Normally, `write(fd, buf, len)` copies your data into the kernel's **page cache** — 4 KB pages managed by the kernel — and the kernel later flushes those pages to the disk controller. With `O_DIRECT`, there is no intermediate copy. The DMA controller on the storage device transfers bytes **directly between your userspace buffer and the disk platters/flash cells**.

This creates a hard hardware constraint: the DMA transfer must be aligned to the device's sector boundary (512 bytes on legacy drives, 4096 bytes on modern drives with Advanced Format). Three things must all be aligned:

1. **The memory buffer address** — the pointer you pass to `write()` must sit at an address that is a multiple of the sector size
2. **The file offset** — the position you seek to must be sector-aligned
3. **The transfer size** — the number of bytes you write must be a multiple of the sector size

Violating any of these produces `EINVAL` from the kernel.

## Why Python's `open()` Can't Satisfy This

Every storage engine in this codebase uses Python's standard `open()`:

- `PageManager.__init__` opens files with `open(file_path, 'r+b')` (`b-tree-storage-engine/btree.py`, line 33 per the PageManager entry)
- Bitcask opens with `"ab"` mode (`hash-index-storage/bitcask.py`, line 68)
- The WAL uses standard buffered file I/O throughout

Python's `open()` does two things that make `O_DIRECT` impossible:

**1. It allocates internal buffers using `malloc()`.** Python's `io.BufferedWriter` (which backs every `open(..., 'wb')` call) maintains an 8 KB write buffer allocated via CPython's general-purpose allocator. `malloc()` guarantees *word alignment* (8 bytes on 64-bit), not *page alignment* (4096 bytes). When you call `f.write(data)`, Python copies your data into this unaligned buffer and eventually calls `write()` with a pointer into it. That pointer will almost never be 4096-byte aligned.

**2. Python `bytes` objects have no alignment guarantees.** When `PageManager.write_page` does `self._f.write(data[:self.page_size])`, `data` is a `bytes` object. CPython allocates `bytes` from its object allocator, which adds a header (ob_refcnt, ob_type, ob_size — about 48 bytes on 64-bit) before the actual byte payload. Even if `malloc` happened to return an aligned address, the header shifts the payload to an unaligned offset.

The O_DIRECT topic entry (`entries/2026/05/29/topic-o-direct-and-dio.md`, lines 42–46) captures exactly this gap — to use `O_DIRECT`, you need:

> - `os.open()` with `os.O_DIRECT` instead of Python's `open()` builtin
> - Aligned memory buffers (e.g., via `mmap` or `ctypes`-allocated memory aligned to 4096 bytes)
> - Sector-aligned offsets — all seeks and write sizes must be multiples of the sector size

## How `posix_memalign` and `mmap` Solve It

**`posix_memalign`** is the POSIX function designed for this exact purpose. It takes an alignment and a size, and returns a pointer guaranteed to be a multiple of the requested alignment:

```c
void *buf;
posix_memalign(&buf, 4096, 4096);  // 4096-byte aligned, 4096-byte buffer
```

In Python, you'd access this through `ctypes`:

```python
import ctypes, ctypes.util

libc = ctypes.CDLL(ctypes.util.find_library("c"))
buf = ctypes.c_void_p()
libc.posix_memalign(ctypes.byref(buf), 4096, 4096)
# buf.value is now a 4096-byte-aligned pointer
# Use os.write(fd, ctypes.string_at(buf, 4096)) for I/O
```

**`mmap`** provides alignment as a side effect. `mmap()` always returns page-aligned memory (the kernel maps virtual pages, which are 4096 bytes on x86/ARM). Python's `mmap` module exposes this:

```python
import mmap
buf = mmap.mmap(-1, 4096)  # anonymous mapping, page-aligned by definition
```

Either approach requires abandoning Python's `open()` and `file.write()` in favor of `os.open()` with `os.O_DIRECT` and `os.write()` with the aligned buffer.

## Why This Codebase Doesn't Bother

The O_DIRECT entry is explicit: **there are zero uses of `O_DIRECT` anywhere in this codebase** (line 10). The `PageManager` already uses 4096-byte pages — the same size as filesystem pages and disk sectors — so it *conceptually* aligns with what `O_DIRECT` requires. But the implementation goes through Python's buffered I/O, and the kernel page cache handles all alignment transparently.

This is the right pedagogical trade-off. `PageManager` teaches the logical structure of a buffer pool (fixed-size pages, free-list allocation, page-number addressing) without the systems-programming complexity of aligned DMA buffers. The `flush()` + `os.fsync()` pattern these engines use (documented across the WAL, B-tree, and Bitcask entries) achieves crash durability through the page cache rather than bypassing it.

A production database like InnoDB would use `O_DIRECT` because it already manages its own buffer pool and doesn't want the kernel duplicating that cache. These reference implementations delegate caching entirely to the kernel — `PageManager` has no application-level page cache, reading from disk on every `read_page` call (per the `btree-page-writes-unbuffered-by-app` belief).

## Topics to Explore

- [function] `b-tree-storage-engine/btree.py:PageManager` — The closest analog to a production buffer pool; its 4096-byte page size matches the alignment `O_DIRECT` would require, making it the natural starting point for adding direct I/O
- [general] `fsync-vs-fdatasync-vs-o-direct` — The three-way trade-off between metadata durability (`fsync`), data-only durability (`fdatasync`), and cache bypass (`O_DIRECT`) that production storage engines navigate
- [function] `write-ahead-log/wal.py:_do_sync` — The central sync dispatch with three modes; understanding why it syncs *through* the page cache clarifies what `O_DIRECT` would replace
- [general] `block-aligned-wal-records` — LevelDB's 32 KB block-aligned log format naturally satisfies `O_DIRECT` alignment constraints; contrast with this WAL's variable-length contiguous packing
- [file] `hash-index-storage/bitcask.py` — Append-only writes with optional fsync; a natural candidate for `O_DIRECT` since each record could be padded to sector boundaries

## Beliefs

- `python-bytes-never-page-aligned` — Python `bytes` objects carry a CPython header (ob_refcnt, ob_type, ob_size) before the payload, so the data pointer is never naturally aligned to 4096-byte boundaries even if the underlying malloc returns an aligned address
- `o-direct-requires-three-alignments` — `O_DIRECT` I/O requires the memory buffer address, file offset, and transfer size to all be multiples of the device sector size (typically 512 or 4096 bytes); violating any one produces `EINVAL`
- `python-open-allocates-unaligned-buffers` — Python's `io.BufferedWriter` (backing `open(..., 'wb')`) uses an internal buffer allocated via CPython's general-purpose allocator, which guarantees word alignment (8 bytes) not page alignment (4096 bytes)
- `mmap-returns-page-aligned-by-definition` — `mmap()` always returns page-aligned memory because the kernel maps at virtual page boundaries, making it a viable source of aligned buffers for `O_DIRECT` without calling `posix_memalign` through ctypes
- `page-manager-4096-coincidence` — `PageManager` defaults to 4096-byte pages matching the OS page and sector size, but this is a coincidental match with `O_DIRECT` alignment requirements — the implementation uses buffered I/O and never enforces alignment

