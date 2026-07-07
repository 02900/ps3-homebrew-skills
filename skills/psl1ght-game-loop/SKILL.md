---
name: psl1ght-game-loop
description: >-
  Game loop & simulation foundations for PS3 homebrew. Use when setting up the main loop,
  exiting cleanly to the XMB, timing the simulation, managing transient objects, embedding
  assets, or adding a debug FPS/CPU/RAM overlay — the per-frame sysUtilCheckCallback XMB-exit
  handshake, fixed timestep, fixed object pools, bin2o asset embedding, and reading frame
  work-time + free memory (PSL1GHT has no CPU-% API; RAM needs a hand-declared syscall).
---

# Game loop & simulation (PS3 / PSL1GHT)

- **Clean XMB exit:** register once with `sysUtilRegisterCallback(SYSUTIL_EVENT_SLOT0, cb, NULL)`,
  then call `sysUtilCheckCallback()` **every frame**; set a `running = 0` flag on
  `SYSUTIL_EXIT_GAME`. Without the per-frame check the console can't reclaim the app (it appears
  to hang on quit).
- **Fixed timestep.** The loop runs at vsync (`tiny3d_Flip` waits for it); use a constant
  `FRAME_DT = 1/60` for rates (speeds, charge/sec, gravity) instead of measuring wall time.
  Simple and deterministic, and it matches the original's tuning for most ports.
- **Fixed pools for transient objects** (projectiles, explosions, coins): a small
  `struct { int active; … }` array with an `active` flag. No per-frame allocation, trivial
  lifetime, cache-friendly.
- **Embed assets with `bin2o`.** Files in `data/*.png|jpg|bin|mod|s3m` become extern symbols
  (`foo_png` / `foo_png_size`) linked into the `.self`, so they work over `ps3load` with **no PKG
  install and no filesystem at runtime**. Name any non-image blob (WAV, level `.txt`, config)
  `*.bin` so the Makefile's `bin2o` rule picks it up. Bulky assets that don't need to be embedded
  can instead go in `pkgfiles/assets/` (PKG only).

## Debug FPS / CPU / RAM overlay

A toggleable stats overlay is cheap and worth having. FPS is trivial (raylib `GetFPS()`); the two
non-obvious pieces:

- **⚠️ Don't use `GetFrameTime()` as a CPU-load metric.** Under `SetTargetFPS`, raylib folds the
  vsync / target wait into it, so it reads ~16.6 ms regardless of actual load. Measure the work
  yourself: `t0 = GetTime()` at the top of the loop, then `work_ms = (GetTime()-t0)*1000` **after
  all `Draw*` calls but before `EndDrawing()`** (that's where the swap + wait happen — draw the
  panel with the *previous* frame's value). `work_ms / 16.67` is the frame-budget load %. A trivial
  2D game reads ~0.2 ms (1%).
- **RAM used/total: PS3 syscall 352, which PSL1GHT doesn't wrap.** Declare it yourself with the
  `lv2syscall1` pattern from `<sys/memory.h>` (in a `.c` TU):
  ```c
  #include <ppu-lv2.h>
  typedef struct { u32 total; u32 avail; } mem_info_t; // bytes
  LV2_SYSCALL sys_memory_get_user_memory_size(mem_info_t *i){ lv2syscall1(352,(u64)i); return_to_user_prog(s32); }
  ```
  `used = total - avail` (shift `>>20` for MiB). A PS3 game gets ~213 MB user memory; guard on
  `total == 0` for emulators that stub the syscall (show `n/a`).
- **There is no OS CPU-% API on PSL1GHT** — nothing in the headers. The frame work-time above is the
  honest processor-load proxy on a fixed-clock console.
