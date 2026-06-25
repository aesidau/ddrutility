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

There is **no test suite, linter, or CI**. "Testing" here means compiling and running a tool against a disk image or ddrescue logfile. `ddru_findbad` is a shell script and is not compiled — `make install` copies `ddru_findbad.sh` to `bin/ddru_findbad`.

## The five tools

- **ddrutility** (`ddrutility.c`) — trivial dispatcher/about program; just prints help/version describing the other tools.
- **ddru_findbad** (`ddru_findbad.sh`) — POSIX shell script. Maps bad sectors from a ddrescue logfile back to filenames. Depends on external tools: `sleuthkit`, `ntfs-3g`, `gnu-fdisk`. Slow; superseded by ddru_ntfsfindbad for NTFS.
- **ddru_ntfsfindbad** (`ddru_ntfsfindbad.c`) — NTFS-native, much faster replacement for ddru_findbad. Parses the MFT directly to map bad ddrescue sectors to files. No third-party deps.
- **ddru_ntfsbitmap** (`ddru_ntfsbitmap.c`) — extracts the NTFS `$Bitmap` and produces a ddrescue *domain* logfile so only used clusters get copied. Invokes the `ddrescue` binary (>= 1.15) as a subprocess.
- **ddru_diskutility** (`ddru_diskutility.c`) — Linux-only low-level disk tool using direct pass-through commands. The whole file is `#ifdef __linux__`; on other platforms `main()` just prints "only works on Linux" and exits.

## Architecture notes

- **Shared code is only between the two NTFS tools.** `ddru_ntfscommon.c`/`.h` is compiled directly into `ddru_ntfsbitmap` and `ddru_ntfsfindbad` (see the makefile rules — `ddru_ntfscommon` is never linked, just included in the compile line). It holds NTFS on-disk structures as `__attribute__((__packed__))` unions (`ntfs_attribute`, `mft_mft`, `boot_sector`, etc.) and shared helpers like `getname()` for UTF-16→UTF-8 filename conversion via iconv. These structs are layered directly over raw disk bytes, so field order/packing must never change casually. `ddrutility.c` and `ddru_diskutility.c` share nothing.

- **Help text is generated, not hand-written in the source.** Each tool has a `<tool>_help.txt` (human-readable) and a corresponding `<tool>_help.h` containing the same text as a C byte array (`unsigned char <tool>_help_txt[]`, xxd-style). The `.c` file `#include`s the `.h` and prints the array for `--help`. **When changing help output, edit the `.txt` and regenerate the `.h`** (e.g. `xxd -i <tool>_help.txt > <tool>_help.h`) — editing the hex array by hand is wrong. Keep the `.1` man pages and the master docs (`ddrutility.texinfo` → `.info`/`.html`/`.txt`) in sync too.

- **Versioning is per-tool and manual.** Each `.c` sets its own `version_number`/`copyright_year` string near the top of `main`'s file. The makefile, README "NEWS" section, and `ChangeLog` track the overall release version separately.

## Conventions

- Targets old-school portability: large-file support via `#define _FILE_OFFSET_BITS 64`, `stdbool`, manual getopt long-option parsing. Keep new code C89/C99-portable and warning-clean under `-Wall -W`.
- GPLv2 header block is at the top of every source file — preserve it on new files.
- Primary dev/test platform is XUbuntu; C tools aim to compile on macOS too, but `ddru_findbad` (shell) is Linux-only in practice.
