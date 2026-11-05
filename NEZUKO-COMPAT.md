# HEXED Resolution 1.1.0 — Nezuko compatibility notes

Target: `SA-MP_Nezuko_v1_0_3 (TESTING)`, package `com.nezukosamp.game`, **armeabi-v7a only**.

## What was wrong (verified against the APK's own libraries)

| # | Finding | Evidence | Fix |
|---|---------|----------|-----|
| 1 | Plugin called `GetAppVersionName`, `GetDisplaySize` (main.cpp) and `CreateDirRecursive` (bridge). They do not exist in Nezuko's AML: its `IAML` vtable has **78 slots (0-77, ends at GetBatteryLevel)**, the vendored header declared 129. Calling them reads past the vtable -> crash on load. | `_ZTV3AML` = 320 bytes in libAML.so | Removed the calls; everything past slot 77 is compiled out (`AML_API_1_4`, never defined) so misuse is a *compile* error. `IAML_VER` is now 1.3.0. |
| 2 | `ICFG` header declared 10 virtuals; Nezuko's `CFG` has **5**. | `_ZTV3CFG` = 28 bytes | Extra methods gated out; unused `Config::Has*` removed. |
| 3 | RsGlobal dimension check ran inside `OnModLoad`, when the renderer is probably not up yet -> permanent "unsupported", no hooks installed. | design review (runtime value NOT observable offline) | Pointer audit stays at load; the *dimension* check is lazy (`COMPAT_PENDING` -> `Recheck()` on game thread). |
| 4 | HUD callback called `LoadConfig()` every frame (~16 allocations + possible INI saves). | code | HUD flag read once. |
| 5 | `Reset` wrote game memory from the bridge worker thread, and was blocked when disabled. | code | Queued to game thread; allowed while disabled. |
| 6 | Custom offset profile required BOTH 32- and 64-bit offsets. | code | Only the current ABI's offset is needed. |
| 7 | Lua hard-coded `com.rockstargames.gtasa`. | manifest | `com.nezukosamp.game` + auto-detect from `getWorkingDirectory()`. |

## Verified OK (no change needed)
* libGTASA.so is 32-bit; GOT slot `0x67910C` is `R_ARM_GLOB_DAT -> RsGlobal` (pin is correct).
* PSDK PLT pins used (`CHud::Draw`, `CTheScripts::Process`, `InitialiseGame`, `CTimeCycle::InitForRestart`) point at those exact symbols; the symbol-based events exist.
* AML vtable slots 0-77 match the vendored header one-for-one.

## New
* `hexed_command()` C export -> Lua calls the plugin **directly via FFI** (file bridge kept as fallback).
* New commands: `info`, `allow <0|1>`. Lua menu rewritten in mimgui (HEXED style), `/hres`.
* Builds `armeabi-v7a` only (the arm64 pin could not be verified).

## NOT verified (needs a device)
* ModInfo field sizes (`amlmod.h`) were not checked against libAML (no disassembler offline). Worst case: AML shows an empty name/version; loading is unaffected because the GUID is at offset 0.
* That writing `RsGlobal.maximumWidth/Height` actually lowers the *rendered* resolution on Nezuko. Use Diagnostics > Probe: if `vp`/`surface` do not change after Apply, the game ignores those fields after init and a framebuffer-level approach would be needed.
* Nothing was compiled with the real NDK here; sources were syntax-checked with stub headers only. Use the GitHub Actions workflow for the real build.
