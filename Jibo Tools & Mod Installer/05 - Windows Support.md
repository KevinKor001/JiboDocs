# Windows Support

This repo supports Windows, but there are two separate “Windows” stories:

1. **Running the Python scripts on Windows** (launchers: `.bat`)
2. **Actually talking to the Jibo over USB in RCM mode** (drivers + build tools)

The USB/RCM part is the trickiest.

## Launchers

- Installer: `../jibo_automod.bat`
  - Runs `python ../jibo_automod.py %*`
  - Warns if not Administrator.

- Updater: `../jibo_updater.bat`
  - Prefers `../.venv/` Python if present.
  - Else prefers `py -3`.

- GUI: `../jibo_gui.bat`
  - Same Python resolution approach.

## Environment setup helper

- `../windows_setup.bat` helps install dependencies via MSYS2 and explains Zadig.

It installs (via MSYS2):

- MinGW gcc/make
- libusb
- ARM toolchain (`arm-none-eabi-gcc`) for payload builds

## USB driver requirement (RCM mode)

When Jibo is in RCM mode it enumerates as NVIDIA APX (`0955:7740`).

On Windows you typically need to replace its driver with WinUSB:

- Use Zadig and install WinUSB for the “APX” device.

Without this, ShofEL won’t be able to talk to the device.

## Editing mode.json on Windows

Linux can mount the ext filesystem image and edit `/var/jibo/mode.json` normally.

Windows usually can’t mount the ext image easily, so `../jibo_automod.py` tries alternative methods:

1. **debugfs (recommended)**
   - If `debugfs.exe` is available on PATH, the tool can edit the file inside the ext filesystem image safely.
   - Install via MSYS2:
     - `pacman -S --needed e2fsprogs`

2. **Raw in-place patch (last resort)**
   - The tool searches the partition image bytes for JSON patterns and overwrites them without changing length.
   - This can fail if the on-disk JSON format differs or if there’s no safe padding space.

If you’re on Windows and you’re seeing warnings about raw edits or the tool can’t find patterns, installing `debugfs` is the best fix.

## WSL option

Another workable approach is to use WSL2 and follow the Linux path.

Caveat: you still need USB passthrough into WSL (which can be non-trivial depending on your setup).

## Common Windows failure modes

- Device not detected / can’t open USB:
  - Wrong driver (use Zadig -> WinUSB)
  - Not Administrator
  - Cable is charge-only

- Build failures:
  - Missing MSYS2 packages
  - ARM toolchain missing (payload `.bin` files won’t build)

- mode.json edit failures:
  - Install `e2fsprogs` and ensure `debugfs` is on PATH

## Related docs

- Installer workflow: [[01 - Installer (How It Works)]]
- CLI flags: [[03 - CLI Arguments]]
