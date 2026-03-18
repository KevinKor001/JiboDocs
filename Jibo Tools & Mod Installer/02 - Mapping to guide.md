# Mapping to guide.md

This page explains how the automated installer in `../jibo_automod.py` relates to the manual steps in `../guide.md`.

## Big picture

`../guide.md` is the “do it by hand” method.

`../jibo_automod.py` is the “do the same steps, but scripted” method.

If something goes wrong in automation, the guide is still valuable because it explains:

- how to confirm RCM mode
- how to interpret partition layouts
- what `/var/jibo/mode.json` does conceptually

## Step-by-step mapping

### Part 1 (Connecting / RCM)

Guide:
- Enter RCM mode and confirm with `lsusb` showing NVIDIA APX (`0955:7740`).

Tool:
- `detect_jibo_rcm()` tries `lsusb` on Linux.
- Falls back to scanning `/sys/bus/usb/devices` if `lsusb` is missing.
- On Windows it can’t reliably detect in the same way, so it prints guidance and proceeds.

Related: [[05 - Windows Support]]

### Part 2 (Build ShofEL)

Guide:
- Clone/build ShofEL manually with `make`.

Tool:
- `build_shofel()` runs `make` inside `../Shofel/`.
- If payload `.bin` files are missing, it requires the ARM toolchain (`arm-none-eabi-gcc`).

### Part 3 (Dump eMMC)

Guide:
- Run: `shofel2_t124 EMMC_READ 0x0 0x1D60000 full_jibo_dump.bin`

Tool:
- Default full workflow calls `dump_emmc()` which runs the same command.
- Fast workflow (`--mode-json-only`) does *not* dump full eMMC; it only reads GPT + `/var`.

### Part 4 (Identify and extract /var)

Guide:
- Run `fdisk -l dump.bin`
- Find partition 5, compute start/count, then `dd` out `var_partition.bin`.

Tool:
- Parses GPT directly (`parse_gpt_partitions()`), with an `fdisk` fallback on Linux.
- Then extracts bytes by seeking directly inside the dump (no `dd` needed).
- Uses heuristics to identify `/var` (partition #5 and roughly 400–600MB).

### Part 4.2 (Edit mode.json)

Guide:
- Mount loop device and edit `jibo/mode.json`.

Tool:
- On Linux: prefers a loop mount and edits `mode.json`.
- On Windows: prefers `debugfs` if installed.
- Raw in-place patch is last resort.

Related: [[01 - Installer (How It Works)]]

### Part 5 (Write /var back)

Guide:
- Convert start sector to hex and run `EMMC_WRITE <start> var_partition.bin`.

Tool:
- Always uses the GPT-derived start sector (so you don’t manually convert it).
- In fast mode, it can patch-write only changed sectors instead of writing the full 500MB.

### Part 6 (Verify)

Guide:
- Read back and compare hashes.

Tool:
- Optional verify step reads back the written range and compares MD5.

## Why the automated tool differs from the guide

The guide is intentionally explicit and teaches the mental model.

The tool tries to reduce “math + manual disk ops” by:

- parsing GPT automatically
- extracting partitions programmatically
- handling mount/debugfs edits
- optionally patch-writing only changed sectors

## When to fall back to the guide

- If your partition layout differs enough that `/var` isn’t partition 5.
- If the filesystem inside `/var` is corrupted and mount/debugfs can’t read it.
- If you want a full backup regardless of speed (the guide defaults to full dump).
