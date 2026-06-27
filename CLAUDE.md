# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

ddrutility is a toolset of standalone C programs (plus one shell script) that aid data recovery alongside GNU ddrescue. There is no library and no shared runtime — each tool is a self-contained command-line program with its own `main()`.

## Build / install / test

```sh
make                 # builds the four C programs (ddrutility, ddru_ntfsbitmap, ddru_ntfsfindbad, ddru_diskutility)
sudo make install    # installs binaries to /usr/local/bin, man pages, and the .info doc
make clean
make uninstall
make ddru_ntfsfindbad   # build a single program
```

Build flags are `gcc -Wall -W -fcommon` (the `-fcommon` matters: the NTFS tools rely on common/tentative globals defined in `ddru_ntfscommon.h`). Link uses `-lm` and the tools also need `libiconv`.

**macOS link caveat:** the NTFS tools call `iconv`/`iconv_open`, which on Linux/glibc are part of libc but on macOS live in a separate `libiconv`. The makefile rules do **not** pass `-liconv`, so `make ddru_ntfsfindbad` / `make ddru_ntfsbitmap` fail at the link step on macOS with "Undefined symbols ... _iconv". This is a known platform limitation, not a code bug. To verify a build on macOS, link by hand, e.g. `gcc -Wall -W -fcommon ddru_ntfscommon.c ddru_ntfsfindbad.c -o ddru_ntfsfindbad -lm -liconv`. The compile of the changed `.c` is what matters for correctness; the `-Wall -W` warnings that predate any change (format `%06ld` vs `suseconds_t`, misleading-indentation in `processdata`/`read_boot_sec_file`) are noise — don't "fix" them as part of an unrelated change.

There is **no test suite, linter, or CI**. "Testing" here means compiling and running a tool against a disk image or ddrescue logfile. `ddru_findbad` is a shell script and is not compiled — `make install` copies `ddru_findbad.sh` to `bin/ddru_findbad`.

## The five tools

- **ddrutility** (`ddrutility.c`) — trivial dispatcher/about program; just prints help/version describing the other tools.
- **ddru_findbad** (`ddru_findbad.sh`) — POSIX shell script. Maps bad sectors from a ddrescue logfile back to filenames. Depends on external tools: `sleuthkit`, `ntfs-3g`, `gnu-fdisk`. Slow; superseded by ddru_ntfsfindbad for NTFS.
- **ddru_ntfsfindbad** (`ddru_ntfsfindbad.c`) — NTFS-native, much faster replacement for ddru_findbad. Parses the MFT directly to map bad ddrescue sectors to files. No third-party deps. By default it assumes a *fully recovered* image and only treats ddrescue's failed statuses (`-`, `/`, `*`) as bad; the `-u`/`--untried` flag also treats non-tried (`?`, uncopied) regions as bad, for use on interrupted/partial rescues. The bad-status test lives in `find_errors()` (the `if (type[x] == ...)` matching each data run against logfile regions); the flag only affects file-data matching, not how non-tried regions in the MFT itself are handled.
- **ddru_ntfsbitmap** (`ddru_ntfsbitmap.c`) — extracts the NTFS `$Bitmap` and produces a ddrescue *domain* logfile so only used clusters get copied. Invokes the `ddrescue` binary (>= 1.15) as a subprocess.
- **ddru_diskutility** (`ddru_diskutility.c`) — Linux-only low-level disk tool using direct pass-through commands. The whole file is `#ifdef __linux__`; on other platforms `main()` just prints "only works on Linux" and exits.

## Architecture notes

- **Shared code is only between the two NTFS tools.** `ddru_ntfscommon.c`/`.h` is compiled directly into `ddru_ntfsbitmap` and `ddru_ntfsfindbad` (see the makefile rules — `ddru_ntfscommon` is never linked, just included in the compile line). It holds NTFS on-disk structures as `__attribute__((__packed__))` unions (`ntfs_attribute`, `mft_mft`, `boot_sector`, etc.) and shared helpers like `getname()` for UTF-16→UTF-8 filename conversion via iconv. These structs are layered directly over raw disk bytes, so field order/packing must never change casually. `ddrutility.c` and `ddru_diskutility.c` share nothing.

- **Help text is generated, not hand-written in the source.** Each tool has a `<tool>_help.txt` (human-readable) and a corresponding `<tool>_help.h` containing the same text as a C byte array (`unsigned char <tool>_help_txt[]`, xxd-style). The `.c` file `#include`s the `.h` and prints the array for `--help`. **When changing help output, edit the `.txt` and regenerate the `.h`** (e.g. `xxd -i <tool>_help.txt > <tool>_help.h`) — editing the hex array by hand is wrong. Note `xxd -i` names the array/length from the input filename, so regenerating in place preserves `<tool>_help_txt[]` / `<tool>_help_txt_len`. Keep the `.1` man pages and the master docs (`ddrutility.texinfo` → `.info`/`.html`/`.txt`) in sync too.

- **Docs are generated from `ddrutility.texinfo`; `.info`/`.txt`/`.html` are build artifacts.** Edit the `.texinfo` source, then regenerate the three outputs with `makeinfo`:
  ```sh
  makeinfo --disable-encoding ddrutility.texinfo -o ddrutility.info
  makeinfo --disable-encoding --plaintext ddrutility.texinfo -o ddrutility.txt
  makeinfo --disable-encoding --no-split --html ddrutility.texinfo -o ddrutility.html
  ```
  `--disable-encoding` is important: the checked-in files were made with an older Texinfo (5.2) that uses ASCII quotes; modern Texinfo (7.x) defaults to Unicode quotes/`©` and produces a noisy whole-file diff without this flag. **Never hand-edit `ddrutility.info`** — it is a binary-ish file ending in a `Tag Table` of byte offsets into each node; editing the body corrupts navigation. Let `makeinfo` rebuild it. The manual's version/date come from `@set VERSION` / `@set UPDATED` near the top of the `.texinfo`; bump those when cutting a release. (These generated docs are often stale in the repo — regenerate them when you touch the source.)

- **The `.1` man pages are help2man-generated from `--version` + `--help`.** The header `.TH`/`NAME` lines and the `COPYRIGHT` section mirror the program's `--version` output. If `help2man` is unavailable, hand-edit the `.1` to match the new `--version` output (version string, date, copyright lines); it will reproduce on the dev box when regenerated.

- **Versioning is per-tool and manual.** Each `.c` sets its own `version_number`/`copyright_year` string near the top of `main`'s file. The makefile, README "NEWS" section, and `ChangeLog` track the overall release version separately. The dispatcher `ddrutility.c` carries the **overall** release version (e.g. `2.8.3`); the NTFS tools carry their own (e.g. `ddru_ntfsfindbad` 1.6). When cutting a release, update: the tool's `version_number`, `ddrutility.c`'s `version_number`, `ChangeLog` (new block at top, dated), README "NEWS" section, and `@set VERSION`/`@set UPDATED` in `ddrutility.texinfo` (then regenerate docs).

- **Copyright attribution.** Scott Dwyer is the original author; do **not** advance his `copyright_year` for changes made by others. The pattern in `version()` is to keep `copyright_year` at Scott's last year and add a separate `int update_year` plus a second `printf ("Copyright (C) %d Andrew E Scott.\n", update_year);` line (and a matching `.br`-separated line in the `.1` man page). The GPL header block at the top of each `.c` is left as-is.

## Conventions

- Targets old-school portability: large-file support via `#define _FILE_OFFSET_BITS 64`, `stdbool`, manual getopt long-option parsing. Keep new code C89/C99-portable and warning-clean under `-Wall -W`.
- GPLv2 header block is at the top of every source file — preserve it on new files.
- Primary dev/test platform is XUbuntu; C tools aim to compile on macOS too, but `ddru_findbad` (shell) is Linux-only in practice.
