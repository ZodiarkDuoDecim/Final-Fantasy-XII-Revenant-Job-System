# ff12-iso-dumper

A tool for unpacking and repacking **Final Fantasy XII (PS2)** ISO images.

It works on both the **IZJS / International Zodiac Job System** image and the original
release. Files are identified by the game's internal *type / index* numbering rather than by
any filesystem, because almost nothing on the disc is actually visible to the DVD filesystem.

### Supported versions

* **Original Japanese** (NTSC-J)
* **PAL** (Europe — English, French, Italian, German, Spanish)
* **NTSC-US** (North America)
* **International Zodiac Job System** (NTSC-J — the IZJS image)
* **Demo**

---

## What it does

FF12 stores its data as a set of *types* (0..36). Each type holds up to a few thousand *files*,
addressed by an index. The disc keeps a **sector table** (at sector 372) that describes, for every
type and index:

* where the file starts (a base sector, in 16-file groups),
* how big it is (compressed sectors + uncompressed size),
* whether it is compressed and with what scheme (LZS, RLE, or "zero").

The game reads all of its data through this table. Only a handful of files (type 0 — the ELF,
IRX drivers, SYSTEM.CNF) are present in the ISO9660/UDF directory; everything else exists purely
as raw sectors behind the table.

The tool:

* **unpacks** an ISO by walking the sector table and writing every file out to a folder,
* **packs** a folder back into an ISO by rebuilding the sector table and writing every file to its
  place.

---

## Building

The project is a Visual Studio C++ solution (`ff12-iso-dumper.sln`, C++20). Build the
`ff12-iso-dumper` project for your platform and you get the `ff12-iso-dumper.exe` binary.

---

## Usage

### Unpack

```
ff12-iso-dumper -u iso_path [out_folder] [-d] [-m=map_file] [-t=N] [-io=N] [-v]
```

Dumps every file from `iso_path` into `out_folder` (defaults to `{iso_path}_unc`). By default
files are copied as-is (still compressed). With `-d` compressed files are decompressed so you can
edit them.

### Pack

```
ff12-iso-dumper -p output_iso [iso_path] dump_folder [-m=map_file] [-t=N] [-io=N] [-v]
```

Rebuilds `dump_folder` into `output_iso`. The original `iso_path` is optional:

* **with** the iso: files missing from the folder are copied from the original, the output keeps
  the original size, and the `start` boot sectors/sector table are taken from the iso directly,
* **without** the iso: every file must be present in the folder, and the folder must contain the
  `start` file (the first 500 sectors with boot data + the sector table, produced by unpacking).

> The **dump folder is self-sufficient**. A full unpack produces everything the packer needs —
> every file plus the `start` file — so you can repack from a dump folder alone, with no original
> ISO, and get a valid image.

### Options

| Option | Meaning |
|--------|---------|
| `-d` | Decompress compressed files during unpack. |
| `-m=map_file` | Resolve real file names from a map file (see below). |
| `-t=N` | Worker thread count. `0`/absent = auto (all cores), `1` = single-threaded. |
| `-io=N` | Max concurrent disk reads/writes. `0`/absent = no cap; lower values help on HDDs. |
| `-v` | Verbose output (per-type / per-file detail). |

### The map file (`-m=map_file`)

A map file is a text list of `type;index;path` lines. It bridges the game's type/index numbering
and real file names, e.g.
`5;824;ps2data\plan_master\us\plan_map\naf_a\naf_a01\global\naf_a01.fpk`.

* During **unpack**, with a map the tool writes files under their real names and paths instead of
  `type_XX\file_IIII`. Without a map you get the raw `type_XX\file_IIII` layout.
* During **pack**, with a map the tool finds each file by its real path. Without a map it looks
  for `type_XX\file_IIII` in the folder.

So a map is the convenient way to work with a human-readable folder; without it the numeric
layout is used for both directions and you can mix them as long as you keep the naming consistent.

---

## Working with only the files you changed

Because packing takes the original ISO when you give it one, you don't need to unpack everything.

```
ff12-iso-dumper -p new.iso original.iso my_edits_folder -m=map.txt
```

`my_edits_folder` can contain **only the files you modified**. Every other file is pulled straight
from the original ISO, already compressed, so nothing needs to be recompressed — this makes the
rebuild very fast. The `start` file is not needed in this mode (the boot sectors come from the ISO).

The same idea works without a map: put your edits as `type_XX\file_IIII`, point at the original
ISO, and everything else comes from it.

### Recommendation

**For modding, the recommended workflow is to always repack against the original ISO and only
place the files you actually changed in the dump folder.** The original ISO acts as the source of
truth for everything you didn't touch, so their compressed sizes stay exactly as the game expects,
the rebuild is fast, and you don't have to recompress the whole game.

That said, a full dump folder is fully self-contained and *can* be repacked on its own (without an
ISO). If you do that, files you leave decompressed get recompressed by the tool; files you keep in
their original compressed form are copied verbatim.

---

## Performance

Both unpack and pack are multithreaded. The worker pool is created once and reused, so creating
thousands of file tasks never spins up a thread per file.

* **Unpack**: reads and (optionally) decompresses files in parallel. A single file is decompressed
  on one thread; I/O concurrency is capped via `-io` so HDDs aren't thrashed by random seeks.
* **Pack**: files are compressed in parallel, but only a small window is held in memory at once.
  Each file is placed in disc order as soon as the files before it finish, written immediately,
  and freed. Large (≥4 MB), copy-only files (movies, etc.) are **streamed** from their source at
  write time rather than buffered whole, so peak memory stays far below the ISO size regardless of
  how big the game is.

Rough figures on an i7-class CPU:

* unpack: 10 seconds to a minute depending on disk speed.
* pack with the original ISO and only a few changed files: 10 seconds to 2 minutes depending on disk speed.
* full repack of every file from a decompressed dump: roughly **30 seconds to 2–4 minutes** depending on disk and CPU speed.

---

## Caveats

* Keep unmodified files compressed (or take them from the original ISO) so their sizes don't
  change. Let the tool recompress files you edited with its compressor rather than pre-compressing
  them yourself.
* Don't pad compressed files with extra data, and don't force a file to a bigger compressed size
  than necessary.
* Files in the folder are matched by name. With a map, that's the real path; without one, the
  `type_XX\file_IIII` naming. Mixing the two consistently is fine, but each file must be findable
  by the chosen scheme.

---

## Why sizes matter — how the game handles memory

The PS2 has **32 MB of RAM**, about **31 MB** of which is usable by games. FF12 loads the
executable and its own data into that space, and with the stack that grows down from the top
of memory there is only about **23–25 MB** left to actually work with for a game this
size. The developers used every trick they could to fit everything in.

One of those tricks is a very memory-conscious way of reading compressed data for some of the files.
The game does not allocate a fresh input buffer and a fresh output buffer for every file. Instead,
most buffers in this game are **static**, or come from an internal allocator (`tkmalloc`) that carves
chunks out of one big static memory block reserved for itself.

When a compressed file is loaded, the game can use a **shared buffer** for both the compressed and
the decompressed data:

```
start of buffer | ... ... ... [compressed data] end of buffer
```

The decompressor writes its output from the *start* of the buffer while it still reads the
compressed bytes sitting at the *end* of the same buffer. Because both pointers move through one
region of memory, the already-decompressed output can overwrite the not-yet-decompressed input
while decompression is still in progress. Whether that happens depends on how big the compressed
data is and how fast the output grows.

**Crash source #1 — compression ratio too low (file too big).** If the compressed data is large,
it stretches far toward the start of the buffer, so the growing output collides with the remaining
compressed bytes early, corrupting them before they are read. The result is garbage data or a
crash.

**Crash source #2 — compression ratio too high (file too small).** This is also a problem, for the
same reason. A very small compressed file leaves a large gap between the start of the buffer and
the start of the compressed data, but decompression *produces* a lot of output from that small
input. The output pointer advances much faster than the input pointer, so it can still catch up to
the input before the input has been fully consumed and overwrite it. Either way the rule is the
same: the compressed size must be close to what the original game data used, and the original
compressor's output sizes are the safe reference point.

This is why you should be careful with file sizes when repacking:

* keep unmodified files compressed (or let them be taken from the original ISO) so their sizes
  don't change,
* let the tool recompress modified files with its reference-matching compressor rather than
  pre-compressing them yourself,
* don't pad compressed files with extra data, and don't force files to a bigger compressed size
  than necessary.

In short: the tool reproduces the original compressor's output, and the game expects sizes that
match that compressor's ratio. The safest repack is one where every file's compressed size matches
the original game's — and that is exactly what this tool produces by default.

This doesn't apply to all the files, but memory limits are tight anyway even in cases of files that
are directly decompressed to their own static buffers.

---

## Compression

Compressed files use the game's LZS format (an 18-byte-window / 2048-byte-distance LZ77 variant)
wrapped in an `ashe` header (magic, size fields, decompressed size, CRC32).

---

## License

See [LICENSE](LICENSE).
