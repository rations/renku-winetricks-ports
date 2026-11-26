# Winetricks for RenkuOS

Recipes that bring [Winetricks](https://github.com/Winetricks/winetricks) to RenkuOS, laid out
like a ports tree so they can move to [RenkuOS/RenkuOSPorts](https://github.com/RenkuOS/RenkuOSPorts).
Winetricks installs the runtimes, fonts and components Windows programs need (Visual C++
runtimes, .NET, core fonts, d3dcompiler, DXVK, vkd3d-proton) into a Wine prefix.

| Port | Version | Notes |
|---|---|---|
| `app-emulation/winetricks` | 20260827~git, REVISION 2 | A snapshot of Winetricks' main branch, below. No patches. |

Planned, in this order:
1. `sys-devel/llvm_mingw`: a MinGW-w64 toolchain on the clang/lld that RenkuOS already ships
   (llvm21 21.1.8), with the mingw-w64 headers and C runtime and libc++ for Windows targets. DXVK
   and vkd3d-proton need it to be built on RenkuOS.
2. `app-emulation/dxvk` and `app-emulation/vkd3d-proton`, built with it, each with a Winetricks
   verb file that installs the packaged DLLs into a prefix.

## Winetricks

**Why a snapshot.** Winetricks releases about once a year; its last release, 20260125, is 44
commits behind its main branch. Those commits include a fix for vcrun2017, vcrun2019 and
vcrun2022 on Wine 11 (`msvcp140_2.dll` was not replaced), new checksums for vcrun2019 and
vcrun2026, and new download addresses for directx and d3dcompiler_47. A packaged Winetricks
cannot use `--self-update`, because a package's files are read-only, so the package has to carry
a recent commit. The recipe pins commit `f3890f670867b5ffbc3938726db45c0f7d16c8ba` (2026-08-27),
and the version is that date, as other snapshot recipes do. The script still reports itself as
`20260125-next`, its own version string.

**Requirements.** Read off the RenkuOS packages: `cmd:wine` and `cmd:wineserver` (Wine), `cmd:7z`
(p7zip), `cmd:cabextract`, `cmd:curl`, `cmd:od` and `cmd:sha256sum` (coreutils), `cmd:tar`,
`cmd:unzip`, `cmd:unzstd` (zstd_bin; the `vkd3d` verb unpacks a `.tar.zst`) and `cmd:xz`.
REVISION 1 lacked `cmd:unzstd`; it was built and tested but never published.

**Known limits on RenkuOS.**
- Command line only. With no verbs, Winetricks starts its GUI only when `DISPLAY` or
  `WAYLAND_DISPLAY` is set (or on macOS), which RenkuOS never sets, and the GUI needs zenity or
  kdialog. Not tried yet.
- RenkuOS has no `pgrep`. Winetricks uses it only in the `icodecs`, `webview2`, `wmi` and
  `fontxplorer` verbs and to stop a download cancelled in the GUI.
- It warns that it runs as root (RenkuOS's user is uid 0) and that `taskset` is missing. Both
  are warnings only.
- The `dxvk` verb installs DXVK's latest release, 3.1.1. A back buffer that a program only
  clears before `Present` shows as never cleared there, on any system: DXVK's bug, fixed by its
  commit 84383189, in no release yet. The planned `dxvk` port carries that fix.

## Tested

REVISION 2, built with `haikuporter` and installed from its package on the RenkuOS build VM
(nightly 95, hrev60072+1180) with Wine 11.22 REVISION 3, 2026-11-26. Three runs, each in a
new prefix (`wineboot -i`), each verb with `winetricks -q`. Every run gave the same results:

| Verb | Result |
|---|---|
| `corefonts` | Installed, about 67 s: 30 fonts in `C:\windows\Fonts`, registered. |
| `vcrun2022` | Installed, 15 to 23 s: the 32- and 64-bit runtimes; `msvcp140_2.dll` is Microsoft's (280,200 bytes), the Wine 11 fix. |
| `d3dcompiler_47` | Installed, 11 to 13 s, both architectures. |
| `win10` | Ran without error, 13 to 16 s. Wine 11 already defaults to Windows 10, so this shows only that a settings verb runs. |
| `dxvk` | DXVK 3.1.1 installed with native overrides, 12 to 15 s. The Wine port's `d3dprobe` then gives 6/8, three times per prefix: D3D9 and D3D11 windows that are only cleared show black, DXVK 3.1.1's bug (above). |
| `vkd3d` | vkd3d-proton 3.1 installed with native overrides, 10 to 14 s. The Wine port's `d3d12probe` then gives 15/15, three times per prefix. |

The same on bare metal (RenkuOS nightly 95, Wine 11.22 REVISION 3), installed from the same
package, which brought in cabextract and zstd_bin from the repositories by itself; three runs,
each in a new prefix, the first one downloading everything afresh: every verb as above,
d3dprobe 6/8 and d3d12probe 15/15 three times per prefix. Slower there: corefonts 108 to 128 s,
vcrun2022 38 to 39 s (239 s with the downloads), the others 26 to 35 s.

Not tried yet: the GUI, other verbs.

## History

| Port | Revision | Date | Change |
|---|---|---|---|
| winetricks | 1 | 2026-11-26 | New. Built and tested on the RenkuOS build VM; never published. |
| winetricks | 2 | 2026-11-26 | Requires `cmd:unzstd`, which the `vkd3d` verb needs. |
