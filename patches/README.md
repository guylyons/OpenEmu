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
- [x] **Controller dead in games (2.5.0)** — fixed: library navigation uses the unhandled-event monitor. Needs a hardware test.
- [x] **Controller A/✕ renamed the game instead of opening it** — focus was on a view inside the grid; navigation now walks the responder chain to the grid and opens the selection.
- [ ] **N64 black screen** when resuming an auto save state. App side fixed (a failed state load now resumes emulation instead of leaving the game stuck); still needs the Restart test below to confirm Mupen64Plus itself renders.
- [ ] **PS2 support** — system plugin now ships in the regular build (moved to "Build SystemPlugins" / "Copy System Plugins to App Plugins"). Has its new sidebar icon; still needs a test with a real game (and a PS2 controller image for Preferences). See below.
- [ ] Gamepad library navigation (AppDelegate) is untested on hardware; no hold-to-repeat yet.
- [ ] Fork the patched cores and commit the fixes there, then repoint the submodules.
- [ ] UME (MAME) is listed in the workspace and Experimental scheme but has no submodule.

## 2.5.0 test findings (2026-10-09)

### Controller doesn't work in games

Root cause: the library-navigation monitor added in d4a638786 uses `OEDeviceManager.addGlobalEventMonitorHandler`.
Any global monitor makes `OEDeviceManager.hasEventMonitor` true, and `AppDelegate.shouldHandleControllerEvents`
returns false while it is (that check exists so key binding in Preferences pauses the game). The monitor is
registered for the app's whole lifetime, so games never get controller input.

Fix (2 lines, `OpenEmu/AppDelegate.swift`, `setUpHIDSupport()`): use the unhandled-event monitor instead, which
doesn't touch `hasEventMonitor`, and drop the `gamepadNavigationMonitor` property:

```swift
// Unhandled, not global: any global monitor sets hasEventMonitor, which stops games receiving controller input.
unhandledEventsMonitor = dm.addUnhandledEventMonitorHandler { [weak self] handler, event in
    self?.navigateLibrary(with: event, from: handler)
}
```

Also: OpenEmu is ad-hoc signed, so the Input Monitoring grant is tied to that exact build. After installing a new build,
run `tccutil reset ListenEvent org.openemu.OpenEmu` and grant it again, or the grant stays with the old copy (it showed
up as "OpenEmu-old" in System Settings).

### N64 (Mupen64Plus 2.5.10) black screen

From the helper logs (`log show --predicate 'process == "OpenEmuHelperApp"'`, Mupen64Plus core messages are `<private>`):

| Game | Auto save load | Rendered | Result |
|---|---|---|---|
| Ocarina of Time (18:12) | `State Load Complete -> 0` (failed) | 0 Metal shader compiles | black |
| Super Mario 64 (18:17) | `State Load Complete -> 0` (failed) | 3 compiles, `bindFBO: OpenGL error 0502` on quit | black |
| Majora's Mask (18:16) | `State Load Complete -> 1` | 30 shader compiles | ok |

So the black screen happens when the auto save state from the previous core fails to load. GLideN64 then never
renders. Quitting saves a new auto save state over the old one anyway, so those auto saves are now probably bad as well.
Next: start the game without resuming the auto save state (Restart) to confirm; if it works, Mupen64Plus should reset
the emulator cleanly when a state load fails instead of leaving a dead video plugin. To see the core's real messages,
turn on private log data (needs a configuration profile on macOS 27).

## 2.6.0 plugin fixes

- Every shipped core now has its system plugin in the regular build. Checked by comparing each core's
  `OESystemIdentifiers` with the bundled plugins' `OESystemIdentifier`: PlayStation 2 (PCSX2), Dreamcast (Flycast) and
  Atari 8-bit (Atari800) were only in "Build Experimental SystemPlugins"; all three now go through
  "Build SystemPlugins" / "Copy System Plugins to App Plugins". Wii is installed by Dolphin's own build phase.
- [ ] Dreamcast sidebar icon is byte-identical to the Arcade cabinet icon (placeholder). Needs Dreamcast artwork.
- [x] PS2 controller image: generated from `patches/assets/ps2-controller-source.png` at the PS1 image's footprint (x 35–465,
  top 153 on the 500 pt canvas) with a matching mask. `Controller-Preferences.plist` positions were rewritten from the
  measured button centres and now use the `OEPS2…` key names; they were copied from PS1 as `OEPSX…`, so no PS2 button
  ever highlighted.
- The OpenEmuKit `fatalError("Audio driver not supported")` is only reachable below macOS 11; the deployment target is
  14.2, so it can't fire. No change needed.

## PS2 support (in progress)

- The core is there: `PCSX2` submodule (OpenEmu/PCSX2-Core), "Build and Install PCSX2" is in both schemes, and
  `PCSX2.oecoreplugin` (1.7.5556, `openemu.system.ps2`) installs to `~/Library/Application Support/OpenEmu/Cores`.
- **The system plugin is missing from the app**: `PlayStation 2.oesystemplugin` (source in
  `OpenEmu/SystemPlugins/PlayStation 2`) is only copied by the "Build & Copy Experimental & Alpha SystemPlugins" target
  (copy phase `94EBBCFB183F42CE00743756`), so the regular "OpenEmu + Cores" build ships without it and PS2 games are
  not recognised. Either build with "OpenEmu + Cores (Experimental, Alpha)" or move the PS2 plugin into the
  "Copy System Plugins to App Plugins" phase like PSP.
- **Needs a PS2 icon.** The icon is a copy of the PS1 icon. Replace the files in
  `OpenEmu/SystemPlugins/PlayStation 2/Images.xcassets` and rename them to match the imageset (update `Contents.json`):
  - [x] `ps2_library.imageset`: done. Generated from `patches/assets/ps2-icon-source.png`: background removed by an
    edge flood fill (keeps the console's own black details), cropped to the console, scaled to 32 px / 16 px wide and
    centred on a transparent square, full colour like the other sidebar icons (they are not templates).
  - `controller_ps2.imageset` / `controller_ps2_mask.imageset`: controls-preferences controller image + mask,
    500×500 and 1000×1000 (@2x). These are the PS1 controller as well.
    **New PS2 controller art for this update:** `patches/assets/ps2-controller-source.png` (DualShock 2). It's the source
    artwork and isn't wired in yet. Scale it into the 500/1000 px canvases, make the matching mask, and check
    the button highlight positions in `Controller-Preferences.plist`, since they were laid out for the PS1 image.
