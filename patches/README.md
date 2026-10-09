# update-cores branch notes

## Building (Xcode 27 / macOS 27 SDK)

```
git submodule update --init --recursive
xcodebuild -downloadComponent MetalToolchain   # once; Xcode 27 ships without it
for p in patches/*.patch; do d=$(basename "$p" .patch); [ -d "$d" ] || d=$(echo "$d" | tr - /); git -C "$d" apply "$PWD/$p"; done
xcodebuild -workspace OpenEmu.xcworkspace -scheme "OpenEmu + Cores" -configuration Release \
  -skipPackagePluginValidation -skipMacroValidation MACOSX_DEPLOYMENT_TARGET=14.2 build
```

- Cores refuse to build in Debug; use Release.
- `MACOSX_DEPLOYMENT_TARGET=14.2`: Xcode 27 rejects targets below 12.0 and most cores target 10.x. 14.2 is the highest any target needs.
- "Build & Install" targets `rm -rf` and replace cores in `~/Library/Application Support/OpenEmu/Cores`. Back that folder up first.

## Patches (submodule fixes not yet committed upstream / to forks)

- `Dolphin-dolphin.patch`: libpng `pngpriv.h` included `<fp.h>` (Classic Mac OS) because `TARGET_OS_MAC` is always defined.
- `Dolphin.patch`: Wii system plugin no longer copies itself into OpenEmu.app (unsigned copy broke app codesign); installed to `~/Library/Application Support/OpenEmu/Systems` by "Build & Install Dolphin" instead.
- `Mednafen.patch`: call `MDFNI_Init()` (once per process) before `MDFNI_InitFinalize()`. The Mednafen 1.32 update dropped it, so no systems were registered and every Mednafen game (PSX, Saturn, PCE, …) aborted on load with `Assertion failed: (MDFNSystems.size())`.
- `Mupen64Plus.patch`: restore `#ifndef IN_OPENEMU` guards around `g_cheat_ctx` and `new_vi` in `main.c` (duplicate symbols with `Compatibility/main.m`).
- `Nestopia.patch`: header paths to `OpenEmu/SystemPlugins` had one `../` too many.
- `OpenEmu-SDK.patch`: when nothing is bound to a controller's left stick, it drives the system's D-pad (first `OEHatSwitchControlsKey` group). Systems whose default mappings bind the stick to their own analog input (N64, PSX, PS2, GameCube, Dreamcast, PSP, Saturn) keep it.
- `OpenEmu-Shaders.patch`: `Compiled.LanguageVersion` threw `unsupportedVersion` for Metal 4.x (the default on macOS 26+), so every shader failed to compile. Treat any version >= 3.0 as 2.4.
- `PPSSPP.patch`: drop AGL.framework link (removed from the SDK; unused); declare `git-version.cpp` as the output of the script that generates it (Xcode 27 failed a clean build with "Build input file cannot be found").

## TODO

- [x] Shaders don't work in the new build — fixed by `OpenEmu-Shaders.patch`.
- [x] Make the left analog stick drive the D-pad by default for all systems where possible — `OpenEmu-SDK.patch`.
- [x] PSX (Mednafen) crashes on every game load — `Mednafen.patch`.
- [ ] Gamepad library navigation (AppDelegate) is untested on hardware; no hold-to-repeat yet.
- [ ] Fork the patched cores and commit the fixes there, then repoint the submodules.
- [ ] UME (MAME) is listed in the workspace and Experimental scheme but has no submodule.
