# Project Architecture Rules (Non-Obvious Only)

- The soft-float/hard-float boundary is the central architectural constraint: `libCTW.so` is armeabi-v7a soft-float, every Linux system library is hard-float. Any new host binding that passes/returns floating point by value needs a `pcs("aapcs")` trampoline; pointers and integers do not.
- Startup order is load-bearing: probe/patch libc time functions → SDL init/window/GL → custom `so_load`/`so_relocate`/`so_resolve` → `patch_openal`/`patch_mpg123`/`patch_opengl`/`patch_game` → `so_initialize` → `jni_init`/`jni_load`. The game's own threads are spawned inside `JNI_OnLoad`'s registered init function.
- The GL context is released from the main thread (`SDL_GL_MakeCurrent(NULL)`) before calling the game init function; the game's render thread must make it current, and `InitEGLAndGLES2`/`makeCurrent` do that on demand.
- `libCTW.so` expects a bionic runtime: 4-byte mutex/cond/sem, `__sF`, `_ctype_`, `__errno`, `__stack_chk_guard`, `__cxa_guard_*`, `pthread_*` semantics. These shims are ABI contracts, not optional helpers.
- Portability floor is glibc 2.31. The `make portmaster` path uses `--sysroot` + `-nostdinc` with careful include ordering specifically to avoid GLIBC_2.34/2.38 and __isoc23_* symbols; changing include/link order silently re-breaks old CFWs.
- Do not bundle SDL2 or OpenAL. They must resolve from the device or PortMaster's armhf runtime; bundled libs sit first in `LD_LIBRARY_PATH`, so anything added there raises the whole port's glibc floor.
- Display ownership is a single decision in the launcher: `OWNS_DISPLAY=1` only for kmsdrm. It gates vtconsole unbind/rebind, `/dev/dri` permissions, tty output, GLES driver pinning, and frontend restart; a compositor path must skip all of it.
- The launcher's EXIT/signal trap is the sole recovery path (stop game, kill gptokeyb, rebind console, restart frontend). It must never `systemctl stop emulationstation` because the launcher runs inside that cgroup.
- Conditional libc patching exists to support one broken Rockchip glibc while not breaking healthy glibc; it is deliberately probe-gated and runs before threads. Do not turn it into an unconditional overwrite.
