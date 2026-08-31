# hyper-kitchen-port

A Windows tool that ports Xiaomi **HyperOS** ROMs from one device to another.

It takes two ROM packages and builds a flashable `super.img` from both:

| ROM | Role | Gives |
|---|---|---|
| **Source** | The new HyperOS you want to run | `system`, `system_ext`, `product`, `mi_ext` |
| **Target** | The ROM that already works on your phone | `odm`, `vendor`, `vendor_dlkm`, `vbmeta` |

The whole job runs in one command: download or pick the ROMs, unpack them, adapt the
source ROM to your device, and repack everything.

---

## Requirements

- Windows (x64). The tool uses bundled `.exe` helpers, so it does not run on Linux.
- Python 3.8 or newer. Only the standard library is used — no `pip install` needed.
- Java, for one migration step (cleaning `MiuiBooster.jar`). Any JDK or JRE on `PATH`
  or in `JAVA_HOME` works. Without Java that single step is skipped and the rest runs.
- Free disk space: about **60 GB**. ROMs are large and get unpacked twice.

---

## Quick start

```bash
git clone https://github.com/Mohammad-Hasan-it-96/hyper-kitchen-port.git
cd hyper-kitchen-port
python XMAPort.py
```

You get a menu:

```
[1] Done Port HyperOS        Full auto workflow (download)
[2] Port from local files    Browse for ROMs already on disk
[C] Open-Source Credits
[D] Clean workspace
```

Pick `[1]` to download the ROMs from the URLs in `config.ini`, or `[2]` to browse for
ROM files you already have. Then enter your device codename (for example `sapphire`)
and confirm.

Results land in `workspace/packed/`.

### Without the menu

```bash
# from URLs
python XMAPort.py --auto --device sapphire --source <URL> --target <URL>

# from local files
python XMAPort.py --auto --device sapphire --source-file D:\roms\src.zip --target-file D:\roms\tgt.zip
```

`--auto` never asks anything, so it suits scripts and CI.

---

## Configuration

Everything lives in `config.ini`. **Save it as GBK**, not UTF-8 — the comments are in
Chinese.

```ini
device_platform=Qualcomm     ; Qualcomm or MTK - must be correct

[source]
url=https://.../source_rom.zip
;file=D:\roms\source.zip     ; local file or folder; wins over url, skips download

[target]
url=https://.../target_rom.zip
;file=D:\roms\target.zip

[packing]
format=erofs                 ; erofs or ext4
compression=lz4hc
compression_level=8
pack_super=true              ; build super.img at the end
device_size=6979321856       ; your device's super partition size, in bytes
enable_adb_debug=true
patch_vbmeta=true            ; disable AVB verification
```

`device_size` must match your phone. A wrong value makes `super.img` fail to build or
fail to flash.

The bottom of `config.ini` holds a `; patch build prop list` section. Every line after
that marker is appended to the target `odm/etc/build.prop`.

### Using local ROMs

Three ways, highest priority first:

1. `--source-file` / `--target-file` on the command line
2. `file=` under `[source]` / `[target]` in `config.ini`
3. Menu option `[2]`, which opens a file browser

A path can be a **ROM archive** or a **folder**. A folder that already holds unpacked
ROM content is used as-is — nothing is copied, since ROMs are several GB.

---

## What it does

| Step | Action |
|---|---|
| 1 | Get both ROMs (download, or use local files) |
| 2 | Extract the archives |
| 3 | Extract partition images from `payload.bin` (also handles block OTA `.dat`) |
| 4 | Unpack the images into file trees |
| 5 | Adapt the source ROM to your device (11 migration steps) |
| 6 | Repack the partitions and build `super.img` |
| 7 | Print a summary |

Step 5 does the real porting work: it renames the device feature file, fixes face
unlock, syncs the VNDK apex, cleans `MiuiBooster`, copies the display and refresh-rate
configs, moves `MiuiCamera` over, patches `build.prop`, and more.

If `super.img` does not fit, the tool automatically removes more preinstalled apps and
retries once.

---

## Troubleshooting

**`[WinError 4551] An Application Control policy has blocked this file`**

Windows Smart App Control blocks the bundled tools because they are not signed. The
tool now checks this before it starts and tells you which file is blocked. To fix:

- Right-click the `.exe` in `tools\` → Properties → tick **Unblock** → OK, or
- Windows Security → App & browser control → **Smart App Control** → Off
  (note: this cannot be turned back on without reinstalling Windows), or
- Set `pack_super=false`. You still get every partition image in `workspace\packed\`.

**`Java runtime not found`**

Install any JDK or JRE, or set `JAVA_HOME`. Only the `MiuiBooster` step needs it; the
other ten migration steps run without Java.

**Not enough space in `super.img`**

Lower `device_size` only if it is wrong for your phone. Otherwise let the automatic
slimming run, or set `is_skip_apex=true` to reuse the source `system_ext` image.

**Logs**

Every run writes `workspace/YYYY-MM-DD-H.log`. Crashes append a full traceback.

---

## Credits

This tool bundles third-party programs. See
[`tools/THIRD_PARTY_NOTICES.md`](tools/THIRD_PARTY_NOTICES.md) for the full list and
their licenses — including 7-Zip, aria2, apktool, payload-dumper-go, erofs-utils, and
the AOSP partition tools.

Press `[C]` in the menu to see the credits in the app.

## License

[Apache License 2.0](LICENSE). This covers the code in this repository. The bundled
third-party binaries keep their own licenses.

## Warning

Flashing a ported ROM can brick your phone. Keep a backup and know how to recover with
fastboot before you flash anything. Use at your own risk.
