# Winetricks for RenkuOS

Recipes that bring [Winetricks](https://github.com/Winetricks/winetricks) to RenkuOS, laid out
like a ports tree so they can move to [RenkuOS/RenkuOSPorts](https://github.com/RenkuOS/RenkuOSPorts).
Winetricks installs the runtimes, fonts and components Windows programs need (Visual C++
runtimes, .NET, core fonts, d3dcompiler, DXVK, vkd3d-proton) into a Wine prefix.

| Port | Version | Notes |
|---|---|---|
| `app-emulation/winetricks` | 20260827~git, REVISION 3 | A snapshot of Winetricks' main branch, below, with one patch for its GUI on RenkuOS. REVISION 3 built; its GUI tried on the build VM, below. The verbs were tested with REVISION 2, which differs only by the patch. |
| `sys-devel/llvm_mingw` | 20251216, REVISION 1 | A MinGW-w64 toolchain on the clang/lld that RenkuOS already ships (llvm21 21.1.8), with the mingw-w64 headers and C runtime, compiler-rt and libc++ for 32- and 64-bit Windows. DXVK and vkd3d-proton need it to be built on RenkuOS. Below. Built and tested on the build VM. |
| `app-emulation/dxvk` | 3.1.1, REVISION 1 | DXVK built with llvm_mingw, with DXVK's fix 84383189, and a Winetricks verb file that installs the packaged DLLs into a prefix. Built and tested on the build VM and on bare metal, below. |
| `kde-apps/kdialog` | kdialog_kf6 26.08.0, REVISION 3 | HaikuPorts' recipe with one patch: `--geometry` works without X11, so Winetricks' windows open at the size it asks for. Replaces the repositories' REVISION 1. Built and tested on the build VM, installed on bare metal; below. |
| `app-emulation/vkd3d_proton` | 3.1, REVISION 1 | vkd3d-proton built with llvm_mingw, with the release's build id, and a Winetricks verb file, as for dxvk. Built and tested on the build VM and on bare metal, below. |


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
- The GUI. With no verbs, Winetricks starts its GUI only when `DISPLAY` or `WAYLAND_DISPLAY`
  is set, or on macOS; RenkuOS never sets them, so up to REVISION 2 it printed its usage. The
  REVISION 3 patch treats RenkuOS as macOS is treated, and starts RenkuOS's Terminal, which has
  no `-e`, with `/bin/sh -c <command>` for the GUI's shell and Wine command items. The GUI
  needs zenity or kdialog; the repositories have only kdialog (`kdialog_kf6`), which this
  package does not require: installing it brings 42 packages (Qt 6, KDE Frameworks 6 and
  their dependencies). Qt draws through its RenkuOS platform plugin, so kdialog's dialogs
  are native windows.
- The repositories' kdialog (`kdialog_kf6` REVISION 1) opens Winetricks' menus at its own
  size, 272 x 264, too cramped to use: it applies `--geometry` only when built with X11, and
  RenkuOS has none. The `kdialog_kf6` here (below) applies it, and the menus open at the
  600 x 400 Winetricks asks for.
- With `--gui` and a verb on the command line, Winetricks shows its warnings (running as
  root, Wine's wow64 mode) as dialogs, one after another, before it starts. Started with no
  verbs, it prints them to its terminal instead.
- RenkuOS has no `pgrep`. Winetricks uses it only in the `icodecs`, `webview2`, `wmi` and
  `fontxplorer` verbs and to stop a download cancelled in the GUI.
- It warns that it runs as root (RenkuOS's user is uid 0) and that `taskset` is missing. Both
  are warnings only.
- The `dxvk` verb installs DXVK's latest release, 3.1.1. A back buffer that a program only
  clears before `Present` shows as never cleared there, on any system: DXVK's bug, fixed by its
  commit 84383189, in no release yet. The `dxvk` port carries that fix.

**The patch** (`patches/winetricks-20260827~git.patchset`) is generated with
`git format-patch` from a branch on the pinned commit, never edited by hand.

## llvm_mingw

[llvm-mingw](https://github.com/mstorsjo/llvm-mingw)'s build scripts at its tag 20251216, which
is made for LLVM 21.1.8, the version of RenkuOS's llvm21. The recipe uses llvm21's clang, lld
and LLVM tools (`--host-clang=clang-21`) and builds only what they lack for Windows targets,
from the LLVM 21.1.8 source (the same tarball as llvm21's) and mingw-w64 at the commit the tag
pins: the mingw-w64 headers, tools (widl, gendef), C runtime (UCRT) and winpthreads,
compiler-rt and libc++, with Control Flow Guard. No sanitizers, OpenMP or LLDB. It installs
into `/boot/system/develop/tools/llvm_mingw`, with the `i686-w64-mingw32-*` and
`x86_64-w64-mingw32-*` commands linked into `/boot/system/bin`.

Before the recipe, the same scripts were run by hand on the build VM (RenkuOS nightly 95,
2026-11-26), about 8 minutes: llvm-mingw's own test programs (34 per architecture, with the
CRT tests: about 2970 checks, no failures) and a C++ test with `std::thread`, mutexes,
condition variables and an exception rethrown across threads, built for both architectures,
ran under Wine 11.22 REVISION 3.

The package, built with `haikuporter` and installed on the same VM (2026-11-26): its 46
commands are in `/boot/system/bin`, it requires only `haiku` and llvm21's `llvm21`,
`llvm21_clang` and `llvm21_lld`, and it has no libtool archives. With the installed commands,
for both architectures: a C hello world, llvm-mingw's `hello-exception.cpp` and the
`std::thread` test above build and run under Wine, exit status 0; `widl` makes a header from
an IDL file that imports `unknwn.idl`.

Programs linked against its libc++ need `libc++.dll` and `libunwind.dll` (in
`develop/tools/llvm_mingw/<arch>-w64-mingw32/bin`) beside them or on `WINEPATH`; otherwise
Wine cannot start them (exit status 53). `-static` links them in, as DXVK's `meson.build` does.

`widl` has the package's directory compiled in, `/packages/llvm_mingw-<version>-<revision>`,
so a new revision is built clean.

## dxvk

DXVK's 3.1.1 tag, its five submodules at the commits the tag pins (one archive each), and
DXVK's commit 84383189 as the patchset, by its author: without it, a back buffer that a
program only clears before `Present` shows as never cleared. Built as DXVK's
`package-release.sh` builds a release, with its own `build-win32.txt` and `build-win64.txt`
cross files, which name the `*-w64-mingw32-gcc` commands that llvm_mingw provides. It installs
the DLLs into `/boot/system/data/dxvk/x32` and `x64`, and `dxvk_renkuos.verb` beside them:

    winetricks /boot/system/data/dxvk/dxvk_renkuos.verb

The verb does what Winetricks' own `dxvk` verb does with a release (copies d3d8, d3d9,
d3d10core, d3d11 and dxgi into the prefix and sets them native), from the package instead of
a download. `winetricks alldlls=default` takes the overrides away again; the DLLs stay in the
prefix, unused.

**Tested.** Built with `haikuporter` on the build VM (RenkuOS nightly 95), 2026-11-26. The
package has the ten DLLs (PE32+ and PE32), which import only Windows system DLLs, the UCRT
`api-ms-win-crt-*` DLLs and each other: the C++ runtime is linked in. Installed from it on the
build VM and on bare metal, with Wine 11.22 REVISION 3 and Winetricks REVISION 2; on each,
three runs, each in a new prefix (`wineboot -i`):

| Step | Build VM | Bare metal |
|---|---|---|
| `winetricks -q /boot/system/data/dxvk/dxvk_renkuos.verb` | Installed, 11 to 12 s | Installed, 26 to 28 s |
| The DLLs in `system32` and `syswow64` | Identical to the package's, 10 of 10 | The same |
| Overrides | d3d8, d3d9, d3d10core, d3d11 and dxgi native | The same |
| The Wine port's `d3dprobe`, three times per prefix | 8/8, through DXVK (its log: "DXVK: 3.1.1"); the 3.1.1 release gives 6/8 | The same |
| Then `winetricks -q alldlls=default`, in the last prefix | No overrides left; `d3dprobe` 8/8 through Wine's wined3d (no DXVK log) | The same |

**Direct3D 12 needs vkd3d-proton beside it.** With DXVK's `dxgi.dll` in a prefix, Wine's own
d3d12 cannot create a device: `D3D12CreateDevice` returns E_NOINTERFACE, and `d3d12probe`
stops there (1 pass, 1 fail). Winetricks' own `dxvk` verb, with the 3.1.1 release, does the same: in one fresh prefix
on the build VM, `d3d12probe` gave 15/15 before it and stopped there after it, and after this
verb too. With vkd3d-proton installed as well, `d3d12probe` gives 15/15: the packaged one
(*vkd3d_proton*, below) or Winetricks' `vkd3d` verb (*Tested*, below).

## vkd3d_proton

vkd3d-proton's 3.1 tag, and the submodules its build uses, at the commits the tag pins (one
archive each): Vulkan-Headers and SPIRV-Headers, dxil-spirv, and dxil-spirv's own SPIRV-Headers
and dxbc-spirv, with that one's SPIRV-Headers. dxil-spirv's SPIRV-Cross and SPIRV-Tools are for
its command-line tools and are not fetched. The port is named `vkd3d_proton`, as a package name
cannot have a hyphen. Built as vkd3d-proton's `package-release.sh` builds a release, with its
own cross files, which name llvm_mingw's `*-w64-mingw32-gcc` and `*-w64-mingw32-widl`. It
installs the DLLs into `/boot/system/data/vkd3d-proton/x86` and `x64`, as the release names
them, and `vkd3d_proton_renkuos.verb` beside them:

    winetricks /boot/system/data/vkd3d-proton/vkd3d_proton_renkuos.verb

The verb does what Winetricks' own `vkd3d` verb does with a release (copies d3d12 and
d3d12core into the prefix and sets them native), from the package. `winetricks
alldlls=default` takes the overrides away again.

**The patch.** vkd3d-proton's build id is `git describe` of its source tree, and a tree
without git, such as a release tarball, gets the fallback `12345678`: every tarball build, of
any version, would have the same id. vkd3d-proton keys the pipeline libraries a program saves
(`ID3D12PipelineLibrary`) on that id, so a library saved by an older build would be taken as
current. The patchset makes the fallback `4522cc1a8b1347f`, the tag's commit cut to 15
digits, the id the official 3.1 DLLs carry (it is in their `d3d12core.dll`). Drop it with the
next release, whose id differs.

**Tested.** Built with `haikuporter` on the build VM (RenkuOS nightly 95), 2026-11-26. The
package has the four DLLs (PE32+ and PE32), which import only Windows system DLLs, the UCRT
`api-ms-win-crt-*` DLLs and dxgi. Installed from it on the build VM and on bare metal, with
Wine 11.22 REVISION 3, dxvk REVISION 1 and Winetricks REVISION 3 (build VM) or 2 (bare metal);
on each, three runs, each in a new prefix (`wineboot -i`), with both packaged verbs:

| Step | Build VM | Bare metal |
|---|---|---|
| `winetricks -q /boot/system/data/dxvk/dxvk_renkuos.verb` | Installed, 5 s | Installed, 10 s |
| `winetricks -q /boot/system/data/vkd3d-proton/vkd3d_proton_renkuos.verb` | Installed, 4 s | Installed, 11 to 12 s |
| The vkd3d-proton DLLs in `system32` and `syswow64` | Identical to the package's, 4 of 4 | The same |
| Overrides | d3d8, d3d9, d3d10core, d3d11, d3d12, d3d12core and dxgi native | The same |
| The Wine port's `d3dprobe`, three times per prefix | 8/8, through DXVK ("DXVK: 3.1.1") | The same |
| The Wine port's `d3d12probe`, three times per prefix | 15/15, through vkd3d-proton (its log: "vkd3d-proton - build: 4522cc1a8b1347f.") | The same |
| Then `winetricks -q alldlls=default`, in the last prefix | No overrides left; `d3dprobe` 8/8 and `d3d12probe` 15/15 through Wine's own DLLs (no DXVK or vkd3d-proton log) | The same |

## Tested

Winetricks REVISION 2, built with `haikuporter` and installed from its package on the RenkuOS build VM
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

**REVISION 3** on the same VM, with `kdialog_kf6` 26.08.0 installed from the repositories:
`winetricks` with no verbs reports "winetricks GUI enabled, using kdialog 26.08.0" and opens
its first menu, a window titled "Winetricks - choose a wineprefix" (visible, checked with
`hey`); closing it ends Winetricks. `winetricks --gui winecmd` starts
`Terminal /bin/sh -c "'wine' 'cmd.exe'"`, and `cmd.exe` runs in it, in the prefix's C:
drive.

On bare metal, tried by hand on its screen, with Wine 11.22 REVISION 3, Winetricks REVISION 3 and the
repositories' `kdialog_kf6` 26.08.0 REVISION 1: the GUI opened in the default prefix, after
Wine updated it; `d3dcompiler_47`, chosen in its menus, installed (the same two DLLs as the
command-line test, byte for byte, set native), and the GUI went back to the prefix's menu.
The windows were too small to use comfortably (272 x 264), which the `kdialog_kf6` port below
fixes. With it installed on the build VM, Winetricks' first window opens at 600 x 400.

Not tried yet: other verbs.

## kdialog

HaikuPorts' `kdialog_kf6-26.08.0.recipe`, unchanged except for its REVISION and the
patchset, so that it replaces the repositories' REVISION 1. kdialog parses `--geometry` with
Xlib's `XParseGeometry` and, built without X11 as on RenkuOS, ignores it: every dialog opens
at its smallest size. The patch parses the same syntax with Qt when there is no X11, resizes
to the size and moves to the offsets given; for a negative offset it counts the window's
border and title bar in, so `-0-0` puts the window in the corner. The X11 build is unchanged.

**Tested** on the build VM (1024 x 768), reading each window's frame with `hey`: no geometry
gives 272 x 264, as before; `600x400+100+100` 600 x 400 with the frame at 100,100;
`800x600`; `+300+200` at the default size; `600x400-0-0` and `600x400-20-10` the frame in
the corner, and 20 and 10 pixels from the right and bottom edges; an invalid geometry is
ignored. Winetricks' first window, "Winetricks - choose a wineprefix", opens at 600 x 400.
Installed on bare metal; nobody has looked at it there yet.

Building it needs the KDE Frameworks 6 and Qt 6 development packages installed on the build
machine (`extra_cmake_modules`, `kdbusaddons6_devel`, `kguiaddons6_devel`,
`kiconthemes6_devel`, `kio6_devel`, `knotifications6_devel`, `ktextwidgets6_devel`,
`kwindowsystem6_devel`, `qt6_tools_devel`, `qt6_base_devel`, `zlib_devel`): haikuporter
takes them from the installed packages.

## Moving to a new release

Each of these patches holds for one release only. When a port moves to a new upstream
version, check them before anything else:

- **vkd3d_proton: drop the patchset** and its `PATCHES` line. The patch sets the build id of the
  3.1 tag, `4522cc1a8b1347f`; kept, it would give the new release 3.1's id, and pipeline
  libraries saved by 3.1 would be taken as current. If tarball builds of the new release still
  fall back to `12345678`, make the patch again with the new tag's commit, cut to 15 digits,
  and check that the release's own `d3d12core.dll` carries that id.
- **dxvk: drop the patchset** once the new release has DXVK's commit 84383189 (the release's
  `git log` contains it); check with `d3dprobe`, which gives 6/8 without the fix.
- **kdialog: keep the patchset** until a kdialog release applies `--geometry` without X11
  (its `src/utils.cpp`), and keep REVISION above the repositories' own.
- **dxvk and vkd3d_proton:** the submodule commits, and their checksums, are the ones the new tag pins; the
  verb files' `title` lines name the version.

## History

| Port | Revision | Date | Change |
|---|---|---|---|
| winetricks | 1 | 2026-11-26 | New. Built and tested on the RenkuOS build VM; never published. |
| winetricks | 2 | 2026-11-26 | Requires `cmd:unzstd`, which the `vkd3d` verb needs. |
| winetricks | 3 | 2026-11-26 | A patch: the GUI starts on RenkuOS, and its terminal items start RenkuOS's Terminal. Built and tried on the build VM. |
| llvm_mingw | 1 | 2026-11-26 | New. Built and tested on the build VM. |
| dxvk | 1 | 2026-11-26 | New. Built and tested on the build VM and on bare metal. (A first build stopped at INSTALL, without a package, because the recipe lacked `ADDITIONAL_FILES` for the verb.) |
| kdialog_kf6 | 2 | 2026-11-26 | The `--geometry` patch, first version: a negative offset left the border and title bar out. Built and installed on the build VM only; never published. |
| kdialog_kf6 | 3 | 2026-11-26 | The patch counts the frame in for a negative offset. Built and tested on the build VM; installed on bare metal. |
| vkd3d_proton | 1 | 2026-11-26 | New, with the build-id patch. Built and tested on the build VM and on bare metal. |
