# Project Documentation Rules (Non-Obvious Only)

- Two launchers exist: root `gtactw.sh` (legacy standalone, fixed `GAMEDIR=/roms/ports/gtactw`, no Wayland detection) and `gta-ctw-portmaster/GTA Chinatown Wars.sh` (the maintained PortMaster launcher; self-locating, Wayland-aware, `GTACTW_DIR`). The regression test only exercises the PortMaster one.
- Names differ by stage: `make` produces `gtactw_r36`; `make portmaster` writes the shipped `gtactw.armhf`, `installer.armhf` and `libclock_fix.so` directly into `gta-ctw-portmaster/gtactw/`. No rename or copy step is needed.
- Installer sentinel mismatch: README says `ROM.WAD`, `GXT.obb.mp3`, and `libCTW.so`; `installer.c` checks all three, but the launcher only shortcut-checks the first two before invoking the installer.
- "Android" is emulated, not containerised: `src/jni_patch.c` fabricates a JNI vtable at hardcoded byte offsets in `fake_env`, and `src/main.c` supplies bionic ABI shims (`__sF`, `_ctype_`, 4-byte pthread objects). There is no Java VM.
- `so_util.c` is a bespoke ELF loader; `libCTW.so` is never `dlopen`ed. `so_symbol()` finds game symbols, while system GL/AL/mpg123 come from `dlopen`/direct linkage.
- OpenAL is statically inside `libCTW.so` with only the Android AudioTrack backend; `openal_patch.c` re-hooks every AL/ALC symbol to system `libopenal.so.1` and stubs `alc_audiotrack_*`.
- `src/opengl_patch.c` contains game-specific hacks, not generic wrappers: BGRA→RGBA swizzle for Mali-400, a p=12 model-matrix translation fix using a captured p=18 view matrix, 4× MSAA hints, mipmap/anisotropic setup.
- The UI/installer font is embedded (`src/installer_font.h`, generated from Terminus, printable ASCII only); there is no SDL2_ttf dependency and installer artwork is extracted from the user's APK at runtime.
- `README.md` (root) is the build/install overview; `gta-ctw-portmaster/README.md` documents device compatibility (glibc 2.31 floor, armhf-only, no bundled SDL2/OpenAL, language and display overrides).
