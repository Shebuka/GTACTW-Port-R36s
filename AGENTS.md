# AGENTS.md

This file provides guidance to agents when working with code in this repository.

## What this is

A 32-bit ARM Android-on-Linux loader: `src/main.c` maps the game's original `libCTW.so` (armeabi-v7a, soft-float) and hooks its JNI/GL/AL/mpg123 imports. It is not a normal Linux app and most changes touch ABI compatibility.

## Commands

- `make` → `gtactw_r36` + `libclock_fix.so`. Expects the armhf sysroot at `armhf-sysroot/root`; CI instead runs `make SYSROOT=/` on a host with armhf dev libs.
- `make portmaster` → old-glibc `gtactw.armhf`, `libclock_fix.so`, `installer.armhf` written directly into `gta-ctw-portmaster/gtactw/`. Run `scripts/build-bullseye-sysroot.sh` first; no rename or copy step is needed.
- [`.github/workflows/build.yml`](.github/workflows/build.yml:1) builds in a Debian 11 container with `make portmaster SYSROOT=/ BULLSEYE=/` (the container is the 2.31 sysroot), stages `gtactw.zip` plus flat update binaries (`gtactw.armhf`, `installer.armhf`, `libclock_fix.so`), and attaches all of them to GitHub releases on `v*` tags.
- `scripts/test-launcher-display.sh [baseline-launcher]` is the only regression suite; it stubs privileged commands and asserts launcher behaviour for kmsdrm/Wayland/override paths.
- `make font` regenerates the committed `src/installer_font.h` from `/usr/share/consolefonts/Lat15-TerminusBold16.psf.gz`; never hand-edit that header.
- No linter/formatter is configured. Do not add one.

## Non-obvious invariants

- Any hook that takes/returns `float`/`double` by value must be declared `__attribute__((pcs("aapcs")))` (see `SOFTFP`/`*_abi` in `src/opengl_patch.c`, `src/openal_patch.c`, `src/mpg123_patch.c`). The game uses soft-float, the system libs hard-float; without the thunk the bug is intermittent (e.g. wrong `AL_PITCH`).
- The engine defaults to logical 640×480. `main.c` derives the physical size from the desktop/GL drawable and computes a centred 4:3 viewport; `opengl_patch.c` remaps `glViewport`/`glScissor`/`glClear` using the engine size (`g_game_*`). `GTACTW_INTERNAL_RES=viewport` is the experimental opt-in that sets the engine size to the 4:3 viewport (e.g. 720×540); the PortMaster launcher reads it from `conf/internal_res.txt`. Do not make `OS_ScreenGetWidth/Height` report the physical size.
- Every symbol imported by `libCTW.so` must be listed in `default_dynlib[]` in `src/main.c`; an unresolved GOT slot becomes a PLT pointer and crashes later.
- Never include `<time.h>` or `<pthread.h>` in `src/clock_fix.c` or `src/clock_preload.c` — glibc's `__asm__` alias creates duplicate `clock_gettime` symbols.
- libc time functions are patched only if a fault probe proves the native implementation broken; don't make this unconditional (causes `SIGILL` on healthy glibc).
- Bionic pthread mutex/cond/sem are 4 bytes vs glibc's 24/48/16; use the `*_fake` wrappers, never raw glibc functions in the dynlib table.
- The `NVThreadSpawnProc` hook uses a hardcoded Thumb offset `load_bias + 0x78efb9`, tied to APK v1.04 `libCTW.so`.
- `GTACTW_DIR` overrides the compiled-in `/roms/ports/gtactw` data path.

## Release constraints

- Never bundle armhf SDL2/OpenAL; they must come from the device or PortMaster's 32-bit runtime.
- Bundled libs sit first on `LD_LIBRARY_PATH`, so their glibc floor must stay ≤ the executable's (bullseye/2.31).
- `libclock_fix.so` is `LD_PRELOAD`ed only into the 32-bit game process, never exported globally.
- The PortMaster launcher must not `systemctl stop emulationstation` (it runs in the frontend's cgroup); restoration is done from the EXIT/signal trap.
