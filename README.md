# Impexp Console App

A twinBASIC port of `impexp.py`, the standalone twinproj/twinpack import/export tool
documented at <https://docs.twinbasic.com/Features/Packages/Import-Export-Tool>.

## Setup

1. Download ImportExport.twinproj or load the Source .twin files into a twinBASIC console app project.
2. In project, select desired build bitness (32-bit recommended for max portability).
3. Click Build.

## Files

| File | Contents |
|---|---|
| `Entry.twin` | Tree node. Arrays are private with accessors. |
| `ByteBuffer.twin` | Append-only output buffer with capacity doubling. |
| `ByteReader.twin` | Bounds-checked little-endian cursor. |
| `modWin32.twin` | Win32 Declares (optional if reference to WinDevLib) |
| `modShared.twin` | UTF-8, path/filesystem helpers, ordinal sort. |
| `modImpExp.twin` | Parser, serializer, import, export. |
| `modSelfTest.twin` | Test suite. |
| `modMain.twin` | CLI, argument splitting, `Say`. |

## Usage

```
impexp import <file.twinproj|.twinpack> [output_dir]
impexp export <input_dir> <output.twinproj|.twinpack>
impexp --self-test [file.twinproj|.twinpack]
```

### Self-test

The sample path is **optional**. Supply one and all ten tests run; omit it and the four
sample-driven tests are skipped and the six synthetic ones still run. A path that is
supplied but doesn't exist is treated as an error, since that's almost certainly a typo.

Any `.twinproj` or `.twinpack` works — just save a new project from the IDE. Note that
the Python original hardcodes a path to `indexer/sample.twinpack` in twinBASIC's own
development repository and exits 1 when it's missing, which is always: that fixture
isn't part of the published single-file download. Its documented `--self-test`
invocation therefore can't succeed as shipped.

Expected output:

```
  [INFO] root "XXXXXXXXX", 6 files, 6 directories, 380511 bytes
  [PASS] Parse sample (invariants + re-parse consistency)
  [PASS] In-memory round-trip (parse -> serialize -> re-parse)
  [PASS] Serializer idempotence (double round-trip)
  [PASS] Disk round-trip (import -> export -> re-import)
  [PASS] Empty project round-trip
  [PASS] Single-file project round-trip
  [PASS] Flags field preserved on round-trip
  [PASS] Unicode names and ordinal ordering
  [PASS] Bad magic rejected
  [PASS] Truncated input rejected
  10/10 tests passed.
```

The sample tests assert **invariants** — non-empty root name, root is a directory,
file and directory counts stable across a re-parse — rather than constants tied to one
particular fixture. The observed shape is reported as `[INFO]`, not asserted.

### What the suite does and doesn't prove

It proves the port is self-consistent: parse, serialize, and disk round-trips all agree,
and the output matches the round-trip behaviour the docs specify (directories before
files, alphabetical within each group, revision `0x0000` for directories and `0x0002`
for files, flags and revision-trailer entries zeroed).

It does **not** prove the IDE accepts the output. Nothing in-process can check that.
Before trusting this on real work: import a project, export it back, and open the result
in twinBASIC. The IDE regenerates the reset metadata on open, so a round-tripped file
should load and build — but your build of the IDE is the only authority on that.

## Things worth knowing

**The magic number.** `0xEA0BA51C` doesn't fit in a signed `Long`, and twinBASIC has no unsigned integer types yet. It's stored as its two's-complement value (`-368335588`). Only equality is ever tested against it, so the sign never matters. If the compiler rejects the hex literal, substitute the decimal.

**Revision is `u64` on the wire, `LongLong` here.** Signed, but it round-trips bit-exact, which is all the format requires.

**Sort order is load-bearing.** `OrdinalCompare` uses `vbBinaryCompare`. If you swap in `vbTextCompare` or `lstrcmpiW`, exports stop being byte-identical to the reference tool for any project with mixed-case or non-ASCII filenames, because Python's `sorted()` orders by code point. `T_UnicodeAndOrdering` guards this — it asserts that `Zebra` sorts before `apple`.

**Errors raised inside a class lose their description.** twinBASIC classes are COM objects, and an error crossing that vtable boundary is flattened to an HRESULT; the caller sees a bare "Automation error". `ByteReader` is the only class here that raises, and it works around this by stashing the message in `mErrorText` (exposed as `ErrorText`) before raising, with `ParseBuffer` catching at the module boundary and re-raising with the text restored. **If you add a raise to `Entry` or `ByteBuffer`, it needs the same treatment** or the user-facing error message silently degrades.

**Exit codes work.** `Halt` uses `ExitProcess`, and the code reaches the shell — verified with `impexp --self-test nosuchfile` followed by `echo %ERRORLEVEL%`, which prints 1. Non-zero on usage errors, unhandled errors, a missing sample, and any failed test; zero otherwise. Safe to gate a CI step on. `ExitProcess` terminates immediately and skips `Class_Terminate`, which is fine here — nothing holds an OS resource at that point.

**32-bit is a sound default.** A 32-bit build runs on x64 via WOW64, and on Windows on ARM the x86 emulator has been available since Windows 10 ARM, whereas x64 emulation needs Windows 11 — so the 32-bit binary is the more portable artifact, not a compromise. Nothing here benefits from 64-bit registers or address space.

**Memory is proportional to file size, not streamed.** The whole file is held in RAM and copied a few times: on import, once by `ReadWholeFile`, again by `ByteReader.Init` (`m = buf` copies, VB arrays don't alias), and again per blob into each `Entry`. Peak is roughly 3x the file size on both import and export. In a 32-bit process, with ~1.5 GB practically usable, that puts the ceiling somewhere near a 350 MB project file — far beyond any real twinproj. If that ever mattered, the remaining easy win is `ByteReader.Init`, which could take a pointer instead of copying.

**`Dir$` is avoided on purpose.** `BuildTree` recurses, and `Dir$` can't have two enumerations in flight. `ListDirectory` completes the `FindFirstFileW` loop before returning, so recursion afterwards is safe.

**`FILETIME` members are `Long` pairs, not `Currency`.** `WIN32_FIND_DATAW` is 4-byte aligned; an 8-byte member risks padding that would shift `cFileName` and give you garbage filenames.

**Zero-length arrays.** VB-family arrays can't have zero elements, so `Content`/`Revisions` are always paired with an explicit `ContentLen`/`RevisionCount`. Never trust `UBound` on them.

## Not yet done

- **Long path support.** Anything beyond ~260 characters fails. The W APIs are already in use, so this is just a `\\?\` prefix applied after `AbsPath`.
- **Reparse points / symlinks** are followed like ordinary directories during export, same as the Python version.
- **The `revisions` array** is preserved on round-trip but never populated on export, matching the reference tool.
