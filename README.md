# ImpExp Console App

A twinBASIC port of `impexp.py`, the standalone twinproj/twinpack import/export tool
documented at <https://docs.twinbasic.com/Features/Packages/Import-Export-Tool>.

## Setup

1. Download ImportExport.twinproj or load the Source .twin files into a twinBASIC console app project.
2. In project, select desired build bitness (32-bit recommended for max portability).
3. Click Build.
4. Open Command line window
5. Cd to Build location.
6. Follow examples in [Usage section](https://github.com/GCuser99/tbImportExport/blob/main/README.md#usage).

## Usage

### Command Line

Use `import` flag to unpack a .twinproj or .twinpack file into a text file directory tree. Conversely, use `export` to pack an unpacked directory tree into a .twinproj or .twinpack file.
```
impexp import <file.twinproj|.twinpack> [output_dir]
impexp export <input_dir> <output.twinproj|.twinpack>
impexp --self-test [file.twinproj|.twinpack]
```
>Note: you can interchange the argument name `export` for `pack` and `import` for `unpack` - these are synonyms that signal the same thing.

### From twinBASIC

ImpExp.exe can also be run from a twinBASIC project using the `Shell` command.
```vba
'The console window stays open (cmd /k) so you can read it - omit the /k to auto-close the command window.
Shell "cmd.exe /k """"" & <path to ImpExp.exe> & """ export """ & <input_dir> & """ """ & <output.twinproj|.twinpack> & """""", vbNormalFocus

'Redirect the output to a log file
Shell "cmd.exe /c """"" & <path to ImpExp.exe> & """ export """ & <input_dir> & """ """ & <output.twinproj|.twinpack> & """ > """ & <log file path> & """ 2>&1""", vbNormalFocus
```
### Self-test

The sample path is **optional**. Supply one and all eleven tests run; omit it and the four sample-driven tests are skipped and the seven synthetic ones still run. A path that is supplied but doesn't exist is treated as an error, since that's almost certainly a typo.

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
  11/11 tests passed.
```

The sample tests assert **invariants** — non-empty root name, root is a directory, file and directory counts stable across a re-parse — rather than constants tied to one particular fixture. The observed shape is reported as `[INFO]`, not asserted.

### What the self-test does and doesn't prove

It proves the port is self-consistent: parse, serialize, and disk round-trips all agree, and the output matches the round-trip behaviour the docs specify (directories before files, alphabetical within each group, revision `0x0000` for directories and `0x0002` for files, flags and revision-trailer entries zeroed).

It does **not** prove the IDE accepts the output. Before trusting this on real work: import a project, export it back, and open the result in twinBASIC. The IDE regenerates the reset metadata on open, so a round-tripped file should load and build — but your build of the IDE is the only authority on that.

## Round-tripping through GitHub

The usual workflow is: `import` a `.twinproj` into a directory tree, commit that tree to a repository, and later `export` it back into a `.twinproj`. This works well, with one thing worth knowing.

**Git does not track empty directories.** If your project contains empty folders — including empty well-known folders such as `Sources`, `Resources`, or `Settings` — they will not survive a push and pull. They exist in your local tree after `import`, but a fresh clone or pull of that repository will be missing them.

This is a Git limitation, not a tool limitation, and in practice it is harmless. `export` packs exactly what is present on disk: if an empty folder was dropped by Git, it simply won't be in the resulting `.twinproj`. When you then open that file, **twinBASIC recreates the folders it expects on its own.** The round trip comes out whole because the IDE reconstructs the missing structure, not because the packed file carried it.

So the practical guidance is short: don't rely on the packed `.twinproj` to preserve empty folders across a Git round trip — rely on the IDE to regenerate them when it opens the project, which it does.

## Project Files

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

## License

MIT © 2026 GCuser99
