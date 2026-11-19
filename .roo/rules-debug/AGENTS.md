# Project Debug Rules (Non-Obvious Only)

- On-device log is `gtactw/gtactw.log` (launcher truncates then tees). The binary writes diagnostics to stderr, and the launcher redirects stderr into that log only after the installer phase; before that installer output is also mirrored to the tty.
- `abort()` and `raise(SIGABRT/SIGSEGV)` from the game are hooked to print a backtrace and then loop forever (deliberate: keeps the device log readable). A "hang" after `ABORT_HOOK`/`RAISE_HOOK` is therefore expected.
- The SIGSEGV/SIGABRT/SIGILL/SIGBUS handler is installed *after* `SDL_Init` on purpose — SDL installs its own handler during init and obscures the fault address. It prints PC/LR/registers, an annotated stack, and `/proc/self/maps`.
- Addresses inside `libCTW.so` are reported as `libCTW.so + 0xOFFSET`; correlate with `objdump -d libCTW.so`. `dladdr` cannot see the custom-mapped module.
- `jni_load()` never returns and the main thread ends in `for (;;) pause();`. Game work runs on libCTW/OpenAL-spawned threads; do not expect a normal main-loop stack.
- The libc `__clock_gettime64` / `__gettimeofday64` / `gettimeofday` trampolines are installed only when a one-shot fault probe (run single-threaded before SDL/threads) proves the native function crashes. Healthy devices must log "native ... works, leaving libc untouched".
- `libclock_fix.so` is the `LD_PRELOAD` workaround for versioned `__clock_gettime64@GLIBC_2.34` consumers (libstdc++/OpenAL). It is intentionally not exported globally: PortMaster's 64-bit helpers reject it with `wrong ELF class: ELFCLASS32`.
- Launcher regressions are reproducible off-device with `scripts/test-launcher-display.sh`; it stubs sudo/systemctl/dri/tty and records the exact privileged action set. Pass an old launcher as `$1` to diff behaviour.
- If patch scripts differ from launcher names, inspect the trace for `SDL_VIDEODRIVER`, `OWNS_DISPLAY`, and `GTACTW_DIR` — the launcher derives these from its own path and from the compositor socket, not from `/roms/ports` alone.
