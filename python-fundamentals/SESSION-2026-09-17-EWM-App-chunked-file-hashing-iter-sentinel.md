# Chunked file hashing with `iter(callable, sentinel)`

**Date:** 2026-09-17
**Project:** EWM-App
**Not a bug** — concept Q&A while reading `_sha256_of()` in `app/engine_service/swmm_model_import_service.py`:

```python
def _sha256_of(path: Path) -> str:
    digest = hashlib.sha256()
    with open(path, "rb") as fh:
        for chunk in iter(lambda: fh.read(1 << 20), b""):
            digest.update(chunk)
    return digest.hexdigest()
```

## What it's for

Computes a SHA256 checksum of a file without loading the whole file into memory — used here to
fingerprint an imported `.inp` file (stored in `swmm_model_imports.sha256`) so re-imports of an
unchanged file are detectable.

## Concepts explained

- **`hashlib.sha256()`** returns a stateful hash object. `.update(chunk)` can be called multiple
  times with successive pieces of data — the final digest is identical to hashing the whole payload
  in one call. This incremental property is what makes chunked reading possible at all.
- **`"rb"` (binary mode) is required, not `"r"`** — text mode lets Python normalize line endings
  (`\r\n` → `\n`) depending on OS, silently changing the byte content and producing a wrong hash.
- **`1 << 20`** is a bit-shift idiom for `2**20` = 1,048,576 bytes = 1 MiB. Written as a shift rather
  than the literal number as a convention for byte-size constants.
- **`iter(function, sentinel)`** — the two-argument form of `iter()`, distinct from the common
  single-argument `iter(some_list)`. It repeatedly calls `function()` (no arguments) and yields each
  result, stopping the moment a result equals `sentinel`. This replaces a manual `while True: ...
  break` loop.
- **Why `lambda: fh.read(1 << 20)` and not `fh.read(1 << 20)` directly**: `iter()`'s first argument
  must be a *callable* it can invoke repeatedly. Passing `fh.read(1 << 20)` directly would call
  `.read()` once immediately and hand `iter()` a fixed bytes value, not a repeatable action. The
  lambda defers the call so `iter()` can re-invoke it each loop iteration, and `fh` remembers its
  read position between calls.
- **`b""` vs `""`**: the sentinel must be bytes (`b""`), matching what `fh.read()` returns at EOF in
  binary mode — a `""` (str) would never compare equal and the loop would never stop.

## Why chunk instead of `fh.read()` once

```python
# Naive alternative — works, but loads the entire file into RAM at once:
def _sha256_of_naive(path: Path) -> str:
    with open(path, "rb") as fh:
        return hashlib.sha256(fh.read()).hexdigest()
```
Fine for small files; risks high memory use or failure on very large `.inp` files. Chunked reading
caps memory to one chunk size (here 1 MiB) regardless of file size.

## How to find this again

grep for `iter(lambda:` or `hashlib.sha256` in this repo; the pattern generalizes to any
"process file/stream in fixed-size pieces" need (checksums, streaming uploads, etc.).
