# ImpExp Console App

A twinBASIC port of `impexp.py`, the standalone twinproj/twinpack import/export tool
documented at <https://docs.twinbasic.com/Features/Packages/Import-Export-Tool>.

## Why use this

The most common use is keeping an ASCII representation of your twinBASIC projects under version control. A `.twinproj` file is binary — opaque to Git, undiffable, and unmergeable — so committing it directly gives you none of what version control is for. Unpacking it into a directory tree of plain text files means Git can show you real diffs and it means your project's history is readable by anything that reads text.

Beyond storage, the tool is well suited to code-management automation, because pack and unpack are ordinary command-line (or in-code) operations you can script. One workflow it enables: keep a master development copy of a project, then unpack it, modify or strip the copy, and repack it for publication — all without touching the dev copy. For example, if your development project contains source helpers, scratch code, or scaffolding you don't want to ship, a script can unpack a fresh copy, remove those pieces, and pack the result into a clean .twinproj for release. The master stays intact; the published artifact is derived from it on demand rather than hand-edited into existence.

## Setup

1. Download ImportExport.twinproj or load the Source .twin files into a twinBASIC console app project.
2. In project, select desired build bitness (32-bit recommended for max portability).
3. Click Build.
4. Open Command line window
5. Cd to Build location.
6. Follow examples in [Usage section](https://github.com/GCuser99/tbImportExport/blob/main/README.md#usage).

## Usage

### Command Line

Use `import` (or `unpack`) to unpack a .twinproj or .twinpack file into a text file directory tree. Conversely, use `export` (or `pack`) to pack an unpacked directory tree into a .twinproj or .twinpack file.
```
impexp import <file.twinproj|.twinpack> [output_dir] [--clean]
impexp export <input_dir> <output.twinproj|.twinpack> [--force]
impexp --self-test [file.twinproj|.twinpack]
```
>Note: you can interchange the argument name `export` with `pack` and `import` with `unpack` - these are synonyms that signal the same thing.

**Options.** Flags may appear anywhere after the verb.

| Flag | Applies to | Effect |
|-----------|-----------------|---|
| clean | import / unpack | Empty the output directory before unpacking, so no files from a previous unpack linger. Refuses to clean a dangerously shallow target (a drive root or a single top-level folder). |
| force | export / pack | Overwrite the output file if it already exists. Without it, export refuses rather than clobber an existing `.twinproj`. |

### Self-test

The sample path is **optional**. Supply one and all 17 tests run; omit it and the four sample-driven tests are skipped and the nine synthetic ones still run. A path that is supplied but doesn't exist is treated as an error, since that's almost certainly a typo.

Any `.twinproj` or `.twinpack` works — just save a new project from the IDE and run the self-test.

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
  [PASS] Long path (>260 chars) round-trip
  [PASS] Settings guard refuses tree with no Settings
  [PASS] Settings guard passes with Settings file present
  [PASS] Export overwrite guard refuses existing file
  [PASS] Export overwrite proceeds with overwrite flag
  [PASS] Clean-first removes stale files
  [PASS] Clean-first depth guard (PathDepth logic)
  17/17 tests passed.
```

The sample tests proves the port is self-consistent: parse, serialize, and disk round-trips all agree, and the output matches the round-trip behavior the docs specify. However, it does **not** prove the IDE accepts the output. Before trusting this on real work: unpack a project, pack it back, and open the result in twinBASIC. The IDE regenerates the reset metadata on open, so a round-tripped file should load and build — but your build of the IDE is the only authority on that.

## Round-tripping through GitHub

The usual workflow is: `import` a `.twinproj` into a directory tree, commit that tree to a repository, and later `export` it back into a `.twinproj`. This works well, with one thing worth knowing.

**Git does not track empty directories.** If your project contains empty folders — such as an empty `Sources` or `Resources` — they will not survive a push and pull. They exist in your local tree after `import`, but a fresh clone or pull of that repository will be missing them.

For most folders this is a Git limitation, not a tool limitation, and it is harmless. `export` packs exactly what is present on disk: if an empty folder was dropped by Git, it simply won't be in the resulting `.twinproj`. When you then open that file, **twinBASIC recreates the folders it expects on its own.** The round trip comes out whole because the IDE reconstructs the missing structure, not because the packed file carried it. So don't rely on the packed `.twinproj` to preserve empty folders across a Git round trip — rely on the IDE to regenerate them, which it does.

### The one exception: Settings

`Settings` file is different, and `export` treats it differently. It holds the project name, references, version, and compile options. If it is missing, the IDE cannot reconstruct that content: the project opens with its source intact but its references, name, and version blanked.

So **`export` refuses to pack a directory tree that has no top-level `Settings` file entry**, rather than produce a misleadingly useless file. It stops before writing anything and reports:

```
error: Refusing to pack: no 'Settings' entry at the top level of "<dir>". The packed
file would open without its references, project name, or version.
```

In practice `Settings` is usually a file (a name with no extension), not a folder, so Git normally carries it fine. This refusal is mainly a guard against a tree where `Settings` genuinely went missing. If you see it after a fresh checkout, restore the `Settings` entry before packing.

## Project Files

| File | Contents |
|---|---|
| Entry.twin | Tree node. Arrays are private with accessors. |
| ByteBuffer.twin | Append-only output buffer with capacity doubling. |
| ByteReader.twin | Bounds-checked little-endian cursor. |
| modWin32.twin | Win32 Declares (optional if reference to WinDevLib) |
| modShared.twin | UTF-8, path/filesystem helpers, ordinal sort. |
| modImpExp.twin | Parser, serializer, import, export. |
| modSelfTest.twin | Test suite. |
| modMain.twin | CLI, argument splitting, `Say`. |

## License

MIT © 2026 GCuser99
