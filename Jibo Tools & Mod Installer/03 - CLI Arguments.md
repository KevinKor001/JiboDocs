# CLI Arguments

This page documents the CLI flags for the two main tools:

- `../jibo_automod.py` (installer/mod tool)
- `../jibo_updater.py` (OS updater)

If you’re using launchers (`.sh`/`.bat`), they forward all arguments through.

## jibo_automod.py

### Modes (mutually exclusive)

- (no mode flag)
  - Full mod workflow (full eMMC dump, extract/modify/write `/var`).

- `--dump-only`
  - Only dump eMMC; don’t modify anything.

- `--write-partition FILE`
  - Write an already-prepared partition image to eMMC.
  - You must tell it where to write using `--start-sector`.

- `--mode-json-only`
  - Fast path:
    1) read GPT (small)
    2) read `/var` only (~500MB)
    3) modify `/var/jibo/mode.json`
    4) write back either a patch (default) or full `/var`

### Common options

- `--dump-path FILE`
  - Use an existing dump file instead of reading from the device.

- `--output FILE` / `-o FILE`
  - Output file for dumps.

- `--start-sector HEX`
  - Start sector for `--write-partition`.
  - Parsed with base autodetect (`0x...` works).
  - Default is `0x7E9022` (but the full/fast workflows usually compute the start sector from GPT).

- `--force-dump`
  - Re-dump even if a dump exists.

- `--rebuild-shofel`
  - Force rebuilding ShofEL (`make clean` then `make`).

- `--skip-detection`
  - Skip USB “is Jibo connected” checks.

- `--no-verify`
  - Skip verification read-back.

### Fast mode options

- `--full-var-write`
  - Only meaningful with `--mode-json-only`.
  - If set: write the full `/var` partition back (slower).
  - If not set: compute sector diffs and only write changed ranges (faster).

### Examples

- Full mod:
  - `python3 jibo_automod.py`

- Dump only:
  - `python3 jibo_automod.py --dump-only -o my_dump.bin`

- Use existing dump:
  - `python3 jibo_automod.py --dump-path jibo_work/jibo_full_dump.bin`

- Fast mode:
  - `python3 jibo_automod.py --mode-json-only`

- Write a prepared partition:
  - `python3 jibo_automod.py --write-partition var_partition.bin --start-sector 0x7E9022`

## jibo_updater.py

The updater assumes you already have SSH access to the robot.

### Required

- `--ip <host>` (alias: `--host`)
  - IP or hostname of your Jibo.

### Connection

- `--user <name>` (default `root`)
- `--password <pass>` (default `jibo`)
- `--ssh-timeout <seconds>` (default `15`)

### Release selection

- `--releases-api <url>`
  - API endpoint for Gitea releases.

- `--stable`
  - Ignore prereleases.

- `--tag <tag>`
  - Install a specific tag instead of “latest”.

### Archive layout

- `--build-path <relative/path>`
  - If the updater can’t find the `build/` folder automatically, specify where it is inside the extracted archive.

### Safety / UX

- `--state-file <path>`
  - Where it remembers the last applied version per host.

- `--force`
  - Re-download and re-install even if the local state says you’re already on that version.

- `--yes`
  - Don’t prompt for confirmation.

- `--dry-run`
  - Download/extract + connect, but don’t upload files and don’t touch mode.json.

### Returning to normal mode

- `--return-normal`
  - After update, set `/var/jibo/mode.json` back to `normal` (no prompt).

- `--no-return-normal`
  - Never prompt and never change mode back.

### Examples

- Update to latest:
  - `python3 jibo_updater.py --ip 192.168.1.50`

- Stable only:
  - `python3 jibo_updater.py --ip 192.168.1.50 --stable`

- Specific tag:
  - `python3 jibo_updater.py --ip 192.168.1.50 --tag v3.3.0`

- Dry run:
  - `python3 jibo_updater.py --ip 192.168.1.50 --dry-run`

More detail: [[06 - Updater (How It Works)]]
