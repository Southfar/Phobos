# Terrain active animation: implementation and validation

Status: source prototype; not compiled with MSVC and not tested in Yuri's Revenge.

## Baseline and build preparation

The feature branch starts from Southfar snapshot `251a545ecd3ba863a006da1da0e91a2a1e64ed80`, imported from upstream `b1dc830570358315273ce3f958a8846c439c1d7e`. The snapshot omitted the YRpp gitlink. This change restores it to the upstream revision `c6c522f77b58508588d9a8652c53356e4aef7a96`.

On Windows with Visual Studio 2022, the v143 C++ x86 toolset, Windows SDK and ATL installed:

```bat
git submodule update --init --recursive
scripts\build_debug.bat
```

The Linux authoring environment has no MSBuild, Windows command interpreter or game installation. The Windows build script could not run. No DLL is supplied. The repository's instruction to verify an MSVC build before committing could therefore not be fulfilled; this is a reviewable source prototype awaiting that gate.

## Design

`TerrainTypeExt` reads `ActiveAnim` from the terrain image's art section. The scenario extension keeps a serialized vector of terrain/animation pointer pairs. Both pointers go through the existing swizzle mechanism. A live terrain pointer with a null animation means the finite animation has finished, preventing recreation after expiration or save/load.

The existing before-all logic hook discovers configured terrain objects independently of the terrain AI layer. A temporary membership set prevents duplicate creation. The set is never iterated for simulation decisions; creation follows terrain-array order. Work is linear in total terrain plus tracked associations on each update. This favors a small, inspectable first implementation over new executable hooks. It needs profiling before use on large forest maps.

Removal is handled through the existing pointer-removal announcement, terrain removal helper and lethal-damage hook. Animation destruction also clears its association. Both pointers are cleared before recursive animation removal. The next scenario update removes dead entries and reconciles limbo. The tree's own SHP animation is unaffected.

Animation weapon zero-damage support is an independent default-off property. It preserves the old behavior unless `Weapon.AllowZeroDamage=true`, a weapon exists, and animation `Damage` is exactly zero. It uses the existing weapon detonation path, preserving projectile/extra-warhead processing. It does not bypass Warhead eligibility checks, change Attached Effect movement rules, or assign environmental house ownership.

## Checks performed

- `git diff --check`: passed.
- Compiled the actual new scenario methods in a temporary C++20 harness with engine stubs: creation, unconfigured terrain, duplicate prevention, finite completion, restored association, limbo/re-entry, non-removal notifications, recursive removal and dead terrain passed. AddressSanitizer and UndefinedBehaviorSanitizer were enabled. Leak detection was disabled because the host's process restrictions prevented LeakSanitizer from running.
- Compiled the actual animation damage scheduling block in a temporary C++20 harness: ten-update zero-damage cadence, nonpositive delay, default-disabled behavior, ordinary positive-damage cadence and once-per-loop gating passed.
- These harnesses do not compile the full DLL, run native engine methods, or validate a real save/load round trip.
- The repository Check Hooks discovery script reported zero new hook declarations and warned `No DEFINE_HOOK or DEFINE_HOOK_AGAIN found in the diff.` Under that skill, further conflict checks stop here. Existing hook declarations, register extraction and return paths are unchanged.
- IDA/disassembly tools are unavailable. Binary instruction boundaries, native animation attachment behavior and engine lifecycle assumptions have not been independently revalidated.

## Required in-game validation

1. Build Debug and Release with MSVC; resolve all compilation errors before treating this as usable.
2. Place one nonanimated, nonflammable tree with a visible finite active animation. Confirm coordinates and one-shot completion.
3. Repeat with a looping animation. Confirm the terrain's own SHP stays motionless.
4. Destroy the tree, including a crumbling variant; remove terrain through construction and scripted removal where supported. Confirm no orphan animation or stale pointer.
5. Save and load during playback, after finite completion, and after removal. Confirm no duplicate, reset or resurrected animation. Start another map afterward to exercise scenario cleanup.
6. Test `Image` aliases, trees on slopes, shroud/reveal, animation chaining and ignited trees. Terrain-owned animations have existing native special behavior that stub tests cannot verify.
7. Test a zero-damage weapon with effects only, then `ExtraWarheads`, against eligible and ineligible units of every relevant house. Confirm no health loss, correct interval and expected ownership filtering. Configure Warheads to permit zero-damage effects as required by the mod.
8. Test stationary-only Attached Effects against moving infantry. This implementation does not solve repeated application during movement.
9. Profile a dense forest and test multiplayer synchronization before distributing a build.

Use fresh games when changing builds; the new serialized fields change save layout. No compatibility claim is made for existing saves.
