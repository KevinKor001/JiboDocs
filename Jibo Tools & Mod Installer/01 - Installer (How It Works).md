# Installer (How It Works)

The repo has a “CLI installer” and an optional GUI wrapper that launches the same Python tools.

The basic mod logic lives in `../jibo_automod.py`

## What the installer actually does

The tool is automating the exact same high-level steps you do manually in [[1. Get your environment ready!]]

1. Build ShofEL (`shofel2_t124`) if it isn’t built yet.
2. Put the Jibo into NVIDIA Tegra RCM mode (USB device `0955:7740`).
3. Read eMMC contents using `EMMC_READ`.
4. Find the `/var` partition (usually GPT partition #5, ~500MB).
5. Modify `/var/jibo/mode.json` from `normal` to `int-developer`.
6. Write the modified `/var` partition back using `EMMC_WRITE`.
7. Optionally verify by reading back and comparing hashes.

After that, Jibo boots to a checkmark and SSH should work:

- `ssh root@<jibo-ip>`
- password: `jibo`

## Launchers (what runs what)

### Linux launcher

- `../jibo_automod.sh` is a Bash launcher
- It does a light dependency check, then prompts:
  - CLI installer (recommended)
  - GUI installer (experimental)

In CLI mode it runs:

- `python3 ../jibo_automod.py <args>`

In GUI mode it:

- creates `../.venv/` if missing
- installs GUI deps from `../JiboTools/JiboTools/requirements.txt`
- launches `../JiboTools/JiboTools/main_panel.py`

### Windows launcher

- `../jibo_automod.bat` just runs `python ../jibo_automod.py <args>`
- It warns if you’re not running as Administrator.

## How it edits mode.json (important detail)

`../jibo_automod.py` tries multiple strategies to edit `mode.json` inside the `/var` filesystem image:

1. **Linux mount (preferred on Linux)**
   - Mounts `var_partition.bin` as a loop device.
   - Edits `jibo/mode.json` (or a couple fallback paths).
   - Unmounts.

2. **debugfs (preferred for Windows if available)**
   - Uses `debugfs` from `e2fsprogs` to read and rewrite the file inside the ext filesystem image without mounting.
   - This is the safest option on Windows if you can install it (see [[05 - Windows Support]]).

3. **Raw in-place patch (last resort)**
   - Searches the binary image for known JSON string patterns like `{"mode": "normal"}`.
   - Overwrites bytes in-place without changing file size.
   - **If the replacement would require growing the data and there is no safe padding, it refuses.**

>[!note]
Because this is a real filesystem image, the mount/debugfs approaches are more reliable than raw patching.

## Installer modes

The CLI tool supports three main workflows:

- **Full mod** (default): full eMMC dump, then extract/modify/write `/var`.
- **Dump-only**: only dumps eMMC (no modifications).
- **Mode-json-only** (“fast mode”): reads only GPT + `/var` and patch-writes only changed sectors.

Details and flags: [[03 - CLI Arguments]]

## Where output goes

By default, the tool uses `../jibo_work/` for working files:

- `jibo_full_dump.bin` (full mode)
- `gpt_dump.bin` (fast mode)
- `var_partition_original.bin` / `var_partition.bin`
- backups, verification reads, and patch chunks

More: [[07 - Working Directory + State Files]]
