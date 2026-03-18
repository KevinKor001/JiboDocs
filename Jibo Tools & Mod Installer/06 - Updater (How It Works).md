# Updater (How It Works)

The updater is a separate tool from the installer.

- Installer: puts Jibo into developer mode by editing `/var/jibo/mode.json` via RCM + eMMC writes.
- Updater: once you have SSH access, it downloads a JiboOs release and uploads a rootfs overlay to the robot over SFTP.

Implementation: `../jibo_updater.py`

## High-level flow

1. Query the releases API (Gitea JSON) and pick a release:
   - latest (default)
   - stable-only (`--stable`)
   - specific tag (`--tag`)

2. Download the chosen release archive.

3. Extract it safely (checks for path traversal in archives).

4. Find the `build/` folder inside the extracted tree.
   - You can override with `--build-path`.

5. SSH into the Jibo using Paramiko.

6. Ensure `/` is writable:
   - tests with a `touch` on `/.jibo_rw_test`
   - if needed, runs `mount -o remount,rw /`

7. Upload the contents of `build/` into `/` using SFTP.
   - Directories are created as needed.
   - Files are uploaded and chmod is best-effort preserved.
   - Symlinks are uploaded as symlinks if possible; otherwise it uploads dereferenced content.

8. Optionally return Jibo to normal mode by changing `/var/jibo/mode.json` back to `normal`.

9. Record the applied tag in a local state file so repeated runs can short-circuit.

## What “build/” means

Releases are expected to contain a “rootfs overlay” folder named `build/`.

That directory typically includes paths like:

- `etc/`
- `opt/`
- `usr/`
- `var/`

The updater essentially copies those files to the same locations on the robot.

## State file

By default, the updater stores last-applied tag per host in:

- `../jibo_work/update_state.json`

If the state file says the host already has the latest tag, it exits unless you pass `--force`.

More: [[07 - Working Directory + State Files]]

## Safety knobs

- It prompts before upload unless `--yes` is passed.
- `--dry-run` downloads/extracts and connects, but does not upload or edit mode.json.

## Returning to normal mode

At the end, the updater can switch the robot back to normal mode:

- `--return-normal` (force it)
- `--no-return-normal` (never prompt / never change)
- otherwise it prompts (unless `--yes` is passed)

Note: returning to normal mode can disable the “checkmark/dev” behavior depending on how JiboOs uses `mode.json`.

## CLI reference

See [[03 - CLI Arguments]] for the complete list and examples.
