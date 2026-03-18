# Working Directory + State Files

Both the installer and updater use `../jibo_work/` as their “scratch + cache” directory.

This is helpful for:

- keeping your large dumps out of the repo root
- making it obvious what can be deleted vs what is precious

## jibo_automod.py outputs

Depending on mode, you may see:

- `jibo_full_dump.bin`
  - The full eMMC dump (~15GB). Created by full workflow.

- `gpt_dump.bin`
  - Small dump (default 4096 sectors / ~2MB) used by `--mode-json-only`.

- `var_partition_original.bin`
  - Original `/var` read from the robot (fast mode).

- `var_partition.bin`
  - Working copy that gets modified.

- `var_partition_backup.bin`
  - Backup copy made before modifications.

- `verify_partition.bin`
  - Read-back buffer used during verification.

- `var_patch_###.bin`
  - Patch chunks produced by sector-diff patch-writing in fast mode.

- `mode.json.original` / `mode.json.modified`
  - Best-effort debug copies of the JSON before/after editing.

### What you should keep

- Keep `var_partition_backup.bin` somewhere safe.
- If you made a full dump, consider archiving `jibo_full_dump.bin` (space permitting).

## jibo_updater.py outputs

The updater uses:

- `../jibo_work/updates/`
  - `downloads/` cached archives
  - `extracted/` extracted trees

And it tracks state in:

- `../jibo_work/update_state.json`

That file maps host -> last applied tag.

## Cleaning up

- Safe-ish to delete: extracted archives and patch chunks (you can always re-download / re-create).
- Be careful deleting: backups and any full dump you still care about.

## Related docs

- Installer overview: [[01 - Installer (How It Works)]]
- Updater overview: [[06 - Updater (How It Works)]]
