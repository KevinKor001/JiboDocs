# Troubleshooting

This page is a practical checklist for common failure points.

If something fails, also see:

- Installer internals: [[01 - Installer (How It Works)]]
- Windows specifics: [[05 - Windows Support]]
- CLI flags: [[03 - CLI Arguments]]
- Updater internals: [[06 - Updater (How It Works)]]

---

## 1) Jibo not found / can’t talk to APX (RCM)

### Symptoms

- Tool prints “Jibo not found in RCM mode”
- `EMMC_READ` / `EMMC_WRITE` fails immediately
- On Linux, `lsusb` does not show `0955:7740`

### Checklist

- Use a known-good **data** micro-USB cable (charge-only cables are common).
- Ensure the robot is actually in RCM:
  - hold RCM button
  - press reset/power
  - release when you get red LED and no normal boot
- Try a different USB port (avoid hubs).

### Linux permissions

- If it works only with `sudo`, consider installing the udev rule:
  - see `../99-jibo-rcm.rules`

### Windows driver

- Install WinUSB for the “APX” device using Zadig.
- Run the tool as Administrator.

---

## 2) ShofEL build issues

### Symptoms

- `make` fails in `../Shofel/`
- Tool complains about missing payload binaries (like `emmc_server.bin`)

### Checklist

- You need normal build tools: `gcc`, `make`, libusb dev headers.
- If payload `.bin` files are missing you also need the ARM toolchain:
  - `arm-none-eabi-gcc`

Notes:

- The tool’s `--rebuild-shofel` forces a rebuild.

---

## 3) Dump is slow / crashes near the end

### Full dump mode

- Full dump reads ~15GB; multi-hour runtime is normal.
- If it fails at ~99%, you often still have most of the content.

### Use fast mode instead

If your only goal is enabling SSH/dev mode, you can usually skip the full dump:

- `--mode-json-only`

This reads only GPT + `/var` (~500MB) and writes back only what changed.

---

## 4) mode.json edit fails

### Why this happens

`/var` is an ext filesystem image. Editing a file *inside* it is easiest on Linux (mount) and harder on Windows.

### What the installer tries (in order)

- Linux: mount loop device and edit `/var/jibo/mode.json`
- Any OS: try `debugfs` if installed
- Fallback: raw byte-pattern overwrite

### Fixes

- On Windows, install `debugfs` via MSYS2:
  - package: `e2fsprogs`
- If raw patching fails:
  - it likely can’t find the exact JSON string pattern, or it would need to grow the data region.
  - installing `debugfs` is usually the right fix.

---

## 5) Fast mode patch-write writes too much (or falls back)

### What patch-write is

In `--mode-json-only` the tool tries to compute which 512-byte sectors changed between:

- `var_partition_original.bin`
- `var_partition.bin`

Then it writes only those sector ranges.

### When it falls back to full `/var` write

It will write the entire partition if:

- the images differ in size (shouldn’t happen in normal flows)
- too many sectors changed
- too many disjoint ranges were detected

If your edit method rewrites a lot of blocks (filesystem reallocation), you may see a full write anyway.

To force full write explicitly:

- `--full-var-write`

---

## 6) Verify step fails (hash mismatch)

### Meaning

- The read-back does not match the local file.

### Common causes

- Unstable USB connection/cable
- Jibo dropped out of RCM mid-process
- Write succeeded but read-back got corrupted

### What to do

- Re-enter RCM and re-run with `--mode-json-only --full-var-write` (simplest “make it match” path).
- Consider disabling verify to proceed if you’re confident the write worked:
  - `--no-verify`

---

## 7) SSH works but updater fails

### Connection failures

- Check Jibo is booted to the checkmark and reachable:
  - port 22 open
  - `root` / `jibo`

### “Rootfs is read-only” issues

Updater tries to remount `/` read-write:

- `mount -o remount,rw /`

If that fails, the updater will refuse to upload.

### Archive layout issues

If the updater cannot find `build/` automatically:

- pass `--build-path` to point at it inside the extracted tree

### Dry-run

To debug without uploading files:

- `--dry-run`

---

## 8) Where to look for logs / outputs

- Installer work files: `../jibo_work/`
- Updater cached downloads/extraction: `../jibo_work/updates/`
- Updater state tracking: `../jibo_work/update_state.json`

See: [[07 - Working Directory + State Files]]
