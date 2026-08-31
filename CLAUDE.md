# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

XMAPort is a **Windows-only, single-machine Android ROM porting tool** for Xiaomi
HyperOS. It takes two ROM zips — a **source** (new HyperOS, gives `system` /
`system_ext` / `product` / `mi_ext`) and a **target** (the ROM that already runs on
your device, gives `odm` / `vendor` / `vendor_dlkm` / `vbmeta`) — and builds a
flashable `super.img` that mixes the two.

There is no package manifest, no test suite, no CI, and no git repo. It is one script
plus a `tools/` folder of vendored `.exe` / `.jar` binaries. Everything runs from the
directory holding `XMAPort.py`.

## Commands

```bash
python XMAPort.py                       # interactive menu (1 = download, 2 = local files, C, D)
python XMAPort.py --auto --device zorn  # non-interactive, for CI / scripted runs
python XMAPort.py --auto --device zorn --source <URL> --target <URL>
python XMAPort.py --auto --device zorn --source-file D:\roms\src.zip --target-file D:\roms\tgt.zip
```

`--source` / `--target` / `--device` override `config.ini` and `workspace/config.txt`.
In `--auto` mode a missing `config.ini` or missing device codename is a hard failure
(exit 1), not a prompt.

### Local ROMs instead of downloading

A ROM can come from disk rather than a URL. Three ways to say so, highest priority first:

1. `--source-file` / `--target-file` on the command line.
2. `file=` under `[source]` / `[target]` in `config.ini`.
3. Menu option `[2] Port from local files`, which opens a file browser.

A local path always wins over `url=` for that side, and its download is skipped. The
path may be an **archive** (extracted into `source_rom/` / `target_rom/`) or a
**folder**. A folder that already holds ROM content (`payload.bin`, `*.img`,
`*.transfer.list`, `*.new.dat*`, searched up to 3 levels deep) is used **in place** —
nothing is copied, because ROMs run to several GB. A folder holding archives instead has
them extracted as usual. Local paths are validated in `check_local_rom()` before Step 1,
so a wrong path fails immediately instead of part-way through.

The browser is `tkinter.filedialog` (standard library). If tkinter is missing, or the
dialog cannot open, `browse_rom_path()` falls back to typing a path. `--auto` never opens
a dialog — use the CLI flags or `config.ini` there.

Individual stages can be run alone — useful when debugging one step without redoing
the whole pipeline:

```bash
python tools/make_hyper.py speed        # run all 11 migration steps
python tools/make_hyper.py sync_fps     # one step only: speed | extreme | clean_apps
                                        # clean_vk | sync_fps | sync_display | sync_camera | sync_apex
python tools/extract_img.py <img> <out_dir>
python tools/pack_partitions.py erofs lz4hc,8 workspace/source_filesystem/system workspace/packed
python tools/check_img_format.py erofs <img>...   # warn-only, always exits 0
python tools/vbmeta_patch.py workspace/packed
python tools/patch_sf.py <libsurfaceflinger.so> --dry-run
```

`make_hyper.py` derives its root from the **parent** of `sys.argv[0]`'s directory, so it
must stay in `tools/` and be invoked as `tools/make_hyper.py` from the project root.

## Pipeline

`one_click_port()` in `XMAPort.py:757` is the whole workflow. Seven steps, each writing
into a fixed `workspace/` subdirectory:

| Step | Does | Reads → writes |
|---|---|---|
| 1 | Download both ROMs (`aria2c`), or skip for a local ROM | URL → `download_source/`, `download_target/` |
| 2 | Extract archives (`7z`), via `extract_rom_input()` | `download_*` or local path → `source_rom/`, `target_rom/` |
| 3 | Extract partition images | `*_rom/` → `source_payload/`, `target_payload/` |
| 4 | Unpack images to file trees | `*_payload/` → `source_filesystem/`, `target_filesystem/` |
| 5 | Migrate (`make_hyper.py speed`) | edits `*_filesystem/` in place |
| 6 | Repack partitions + `lpmake` | `*_filesystem/` → `packed/` |
| 7 | Print summary + ROM info | reads `build.prop` |

Step 3 auto-detects three ROM shapes (`detect_rom_format`): A/B OTA `payload.bin`
(→ `payload-dumper-go`), block OTA `.dat` / `.dat.br` (→ `extract_dat.py`), or loose
`.img` files (copied as-is). Steps 3 and 4 are skipped when the output already holds
more than 6 files, so re-runs are cheap.

Step 2 returns the directory Step 3 must read (`src_rom_dir` / `tgt_rom_dir`). That is
normally `source_rom/` / `target_rom/`, but for a local already-extracted folder it is
the user's own folder. Use those variables, not the `SRC_ROM` / `TGT_ROM` constants,
anywhere after Step 2.

Logs go to `workspace/YYYY-MM-DD-H.log` (one per hour). Crashes append a full traceback
through the `sys.excepthook` set in `crash_report`.

### Which partition comes from where

This mapping is the core of the tool and is easy to get wrong:

- `system`, `system_ext`, `product` — **repacked from source** filesystem
- `odm` — **repacked from target** filesystem
- `mi_ext` — **copied** from source payload, not repacked
- `vendor` — copied from target payload on Qualcomm; **repacked** from target
  filesystem on MTK (`device_platform=mtk`)
- `vendor_dlkm` — copied from target payload when present
- `vbmeta*` — copied from target payload, then AVB-patched

`unpack_all_img` only unpacks `system system_ext product odm mi_ext`. MTK vendor
repacking therefore has to unpack `vendor.img` on demand in the middle of Step 6.

If `lpmake` reports "not enough space", `create_super_img` returns 1 and the pipeline
automatically runs `make_hyper.py extreme` (removes more preinstalled apps), repacks
`product`, and retries once.

## Directory contract for unpack / repack

`extract_img.py` and `pack_partitions.py` share one flat layout. Breaking it breaks
repacking silently:

```
work/                                  # parent of the partition dir
work/{name}/                           # partition file tree
work/config/{name}_fs_config           # uid/gid/mode per path
work/config/{name}_file_contexts       # SELinux labels
```

`pack_partitions.py` **refuses to pack** when `fs_config` or `file_contexts` is missing —
it will not fall back to default permissions. It rewrites both into `_fixed_{name}_*`
files first: dropping entries for files that no longer exist, and filling in entries for
files added during migration. Missing permissions are guessed from `DEFAULT_PERMS` plus a
majority vote over same-directory / same-extension peers. Before/after dumps land in
`work/config/{name}_fs_config.log`.

Two editable allowlists are auto-generated on first run in `work/config/`:
`fs_special.conf` (paths forced executable, e.g. `bin/su`) and `fc_special.conf` (paths
forced to a specific SELinux label). SELinux auto-completion runs only for `product` and
`system_ext` (`_FC_ALLOW_PARTS`).

ext4 images are read by a pure-Python parser in `tools/zero/` (no 7z), and that parser is
what generates `fs_config` / `file_contexts` on extract. erofs uses `extract.erofs.exe`.

## make_hyper.py — the migration steps

Eleven ordered steps (`run_speed_pipeline`, `tools/make_hyper.py:1414`). A failing step
logs and the pipeline **continues**; the failure count becomes the exit code. Most steps
edit `workspace/source_filesystem/`. `patch_build_prop` is the exception — it appends to
`workspace/target_filesystem/odm/etc/build.prop`.

Things worth knowing before touching it:

- Device codename comes from `workspace/config.txt` (`TARGET_DEVICE=`), written by
  `XMAPort.py`. `sync_features` renames the source `device_features/*.xml` to
  `<codename>.xml`.
- `detect_os_version` reads `ro.mi.os.version.code` from `mi_ext`, falling back to
  `product`. `patch_build_prop` adds Vulkan / skiavk props only when that value is OS4.
- `clean_miui_booster` needs Java: bundled `tools/jre/bin/java.exe`, then `JAVA_HOME`,
  then `java` on PATH, then a scan of `C:\Program Files\Java` and similar. It shells out
  to `tools/apktool.jar`.
- `patch_surfaceflinger` runs only when `device_platform=mtk`. It delegates to
  `patch_sf.py`, which locates `HWComposer::getModesFromLegacyDisplayConfigs` in the ELF
  symbol table and pattern-matches instructions — no hardcoded offsets. Its Chinese
  output is redirected to `workspace/sf_patch.log` and only an English summary is
  printed. `make_hyper.py` detects the "already patched" case by matching the literal
  Chinese string `已是补丁状态` in that output.

## config.ini

One file drives everything. It is **GBK-encoded** with Chinese comments — read and write
it as `gbk`, never `utf-8`, or the comments corrupt.

`read_config()` parses the `[source]` / `[target]` URLs and `[settings]`.
`read_packing_config()` is a **separate, flat parser**: it ignores section headers
entirely and picks up any `key=value` whose key matches its defaults dict. That is why
`device_platform=` sits above `[source]` and still works, and why adding a new packing
key means adding it to that defaults dict.

The trailing `; patch build prop list` marker is parsed by `make_hyper.py`
`patch_build_prop`: every line after it, up to the next `[section]`, is appended verbatim
to the target odm `build.prop`.

Keys that change behaviour most: `device_platform` (qualcomm / mtk), `format`
(erofs / ext4), `pack_super`, `is_skip_apex` (skip repacking `system_ext` and copy the
source image instead), `enable_adb_debug`, `patch_vbmeta`, `device_size` (must match the
real device's super partition), `erofs_old_kernel`.

Step 6 passes packing options to `pack_partitions.py` through environment variables. Two
of them look alike but do different things:

- `XMAPORT_EROFS_LEGACY` — from `erofs_old_kernel`. Adds `-E legacy-compress` to the
  **normal** `mkfs.erofs`, changing the compression format for old kernels.
- `XMAPORT_USE_LEGACY_EROFS` — from `detect_legacy_erofs_marker()`, which returns true
  when the source `product/etc/build.prop` has both `V13` and `DEV` in
  `ro.product.build.version.incremental`. Swaps the **binary** for the older
  `tools/erofs-utils-cygwin/mkfs.erofs.exe` (erofs-utils 1.4).

## Failure handling

Two failure modes bit a real run and are now guarded. Keep the guards when editing.

**Blocked binaries.** The vendored `.exe` files are unsigned, so Windows Smart App
Control / WDAC can refuse to start them — `OSError [WinError 4551]`. This is per-file:
on one machine `img2simg.exe`, `lpunpack.exe`, and `lpmake.exe` were blocked while every
other tool ran. `tool_status()` therefore *probes* each `.exe` (`probe_tool` runs it with
`--help` and only cares that it started), marks blocked ones `[BLOCKED]`, and returns
`{name: error}`. `one_click_port` then aborts before Step 1 — but only for tools this run
actually needs, per `required_tool_names()`. With `pack_super=false`, a blocked
`lpmake.exe` is just a note, because nothing calls it. Every `subprocess` call to a
vendored binary also catches `OSError` and routes it through `report_exec_error()`;
without that, a block surfaces as a crash after ~20 minutes of unpacking.

**Incomplete bundled JRE.** `tools/jre/` here ships `bin/java.exe` but **no `jvm.dll`**,
so it fails with "missing `server' JVM". `find_java_path()` must therefore verify a
candidate with `java_works()` (a real `java -version` run) rather than trusting
`os.path.exists`. It walks bundled → `JAVA_HOME` → `PATH` → `C:\Program Files\Java`, and
falls through to the next candidate on failure. `apktool.jar` here is 3.0.2 and runs on
Java 8, so a system JRE is a fine fallback.

## Known inconsistencies

Real mismatches in the current code — check these before assuming a config flag works:

- `XMAPort.py:927` exports `XMAPORT_IS_SKIP_APEX`; nothing reads it. That flag is applied
  directly in Python instead.
- `XMAPort.py` reads `config.ini` as GBK; `make_hyper.py patch_build_prop` reads the same
  file as UTF-8 with `errors="ignore"`.
- Step labels 4–6 in the printed workflow banner all say "Extract archivesSource".

## Conventions

- Output language is split by layer: `XMAPort.py` and `make_hyper.py` print English
  (`[INFO]` / `[ERROR]` with ANSI colors); `pack_partitions.py`, `extract_img.py`,
  `extract_dat.py`, and `patch_sf.py` print Chinese. Keep each file in its own language.
- Sub-tools are invoked with `sys.executable` and `PYTHONIOENCODING=utf-8`, so Chinese
  output survives a cp1252 console.
- Only the standard library is used. Do not add third-party imports — there is no
  `requirements.txt` and the tool is meant to run on a bare Python install.
- `AUTO_MODE` must stay honoured in anything interactive: `pause()` does nothing when
  stdin is not a TTY, and `prompt()` exits cleanly on EOF.
- The binaries in `tools/` are Windows `.exe` files, which is why this does not run on
  Linux even though the Python is mostly portable. Attributions live in
  `tools/THIRD_PARTY_NOTICES.md`.
