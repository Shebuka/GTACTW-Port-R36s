# Project Coding Rules (Non-Obvious Only)

- Hooks on `libCTW.so` must bridge soft-float → hard-float for every by-value `float`/`double`: declare the hook `__attribute__((pcs("aapcs")))` and call the real symbol from inside it. Pointer-only signatures (`alSourcefv`, `glUniform4fv`) are already safe; do not add a thunk.
- New imported symbols belong in `default_dynlib[]` in `src/main.c`; a missing entry leaves the game's GOT pointing at a PLT stub, not NULL, so it fails much later. `make check-syms` lists undefined imports.
- Use `pthread_*_fake`/`sem_*_fake` wrappers in the dynlib table. Bionic's 4-byte mutex/cond/sem objects cannot hold a glibc object directly; the fakes store a heap pointer in the game's slot.
- `src/clock_fix.c` deliberately does not include `<time.h>` or `<pthread.h>` (the latter pulls in `<time.h>`); glibc's `__asm__` alias would create duplicate `clock_gettime` definitions. Keep new declarations manual, as done for `pthread_exit`.
- The two clock fixes are different mechanisms: `clock_fix.c` defines executable-global overrides/Thumb trampoline targets; `clock_preload.c` is a `-nostdlib`, version-scripted DSO exporting `clock_gettime@@GLIBC_2.4` and `__clock_gettime64@@GLIBC_2.34`.
- `patch_game()` relies on Itanium-mangled names for C++ methods (e.g. `_Z15OS_ThreadLaunchPFjPvES_jPKci16OSThreadPriority`); verify with `objdump`/`nm` before changing a signature.
- `src/so_util.c` is a custom ELF loader (mmap + manual relocations), not `dlopen`; `so_symbol` falls back to a linear dynsym scan, and `dladdr` cannot resolve addresses inside `libCTW.so` — the crash handler special-cases the load bias.
- Bundle/data directory naming is load-bearing: default `make` emits `gtactw_r36` (used by root `gtactw.sh`), while `make portmaster` writes the shipped `gtactw.armhf`, `installer.armhf` and `libclock_fix.so` directly into `gta-ctw-portmaster/gtactw/`.
- Keep the engine at logical 640×480. `main.c` derives `g_phys_*`/`g_view_*` from the SDL desktop mode and GL drawable; `opengl_patch.c` remaps `glViewport`/`glScissor`/`glClear` into the centred 4:3 viewport. `OS_ScreenGetWidth/Height` must stay logical, not physical.
- The installer links SDL2 only but still needs EGL at window creation (SDL2 KMSDRM has no plain framebuffer path); the launcher exports `SDL_VIDEO_EGL_DRIVER`/`SDL_VIDEO_GL_DRIVER` before invoking it.
- Launcher ownership gate: only `OWNS_DISPLAY=1` (kmsdrm) may touch `/dev/tty*`, `/dev/dri/*`, or vtconsole bindings. Wayland/X11 paths must leave them alone.
