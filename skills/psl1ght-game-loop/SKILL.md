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
  to hang on quit). (raylib's `EndDrawing`/`WindowShouldClose` already pump this for you.)
- **⚠️ sysutil modal dialogs (Saved Data Utility, message dialogs) must run on a background
  thread.** `sysSaveListSave2`/`Load2` (and `msgDialog*`) **block** until the user dismisses the XMB
  overlay, but that overlay is driven by `sysUtilCheckCallback()` — so if you call them inline you
  deadlock (the loop that pumps callbacks is stuck in the blocking call). Spawn a `sysThreadCreate`
  worker that runs the blocking call while your **main loop keeps rendering + pumping
  `sysUtilCheckCallback()`**, and poll a `volatile` status flag the worker sets on completion. The
  save dialog needs a `sysMemContainerCreate` (~5 MB) and three callbacks (list → offer a new-save
  slot; status → PARAM.SFO title/subtitle/detail + recreate mode; file → write/read ICON0 + your data
  blob). Return codes: `SYS_SAVE_RETURN_DONE` / `_CANCELED`. Link `-lsysutil` (see the psl1ght-build
  skill's SPRX note). Serialize your own save blob (magic+version, fixed-size fields — same-endian
  on PPU, no swaps); a chess port packed the full position + move history and resumed exactly.
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

A toggleable stats overlay is cheap and worth having. The RAM and no-CPU%-API points below are
**renderer-agnostic** (any PSL1GHT project). The frame-timing helpers named are **raylib-specific**
(`GetFPS`/`GetFrameTime`/`GetTime`/`SetTargetFPS`/`EndDrawing` exist only in the raylib variant) — on
tiny3d / rsxgl you count frames yourself and read a system timer; the *principle* still holds.

- **⚠️ Measure CPU work-time, not total frame time.** The buffer-flip call blocks on vsync, so timing
  across it just measures the wait. Sample from the top of the loop to **just before the flip**:
  `t0 = clock()` at loop start, `work_ms = (clock() - t0) * 1000` right before the present call
  (raylib `EndDrawing()`, tiny3d `tiny3d_Flip()`, rsxgl the egl/RSX flip); draw the panel with the
  *previous* frame's value. `work_ms / 16.67` is the frame-budget load %; a trivial 2D game reads
  ~0.2 ms (1%). **raylib note:** `GetFrameTime()` is useless as a load metric — under `SetTargetFPS`
  it folds in the wait and reads ~16.6 ms regardless — so use `GetTime()` deltas as above. `GetFPS()`
  gives FPS directly; on other renderers derive it from a 1-second frame counter.
- **RAM used/total: PS3 syscall 352, which PSL1GHT doesn't wrap** (works on every renderer). Declare
  it yourself with the `lv2syscall1` pattern from `<sys/memory.h>` (in a `.c` TU):
  ```c
  #include <ppu-lv2.h>
  typedef struct { u32 total; u32 avail; } mem_info_t; // bytes
  LV2_SYSCALL sys_memory_get_user_memory_size(mem_info_t *i){ lv2syscall1(352,(u64)i); return_to_user_prog(s32); }
  ```
  `used = total - avail` (shift `>>20` for MiB). A PS3 game gets ~213 MB user memory; guard on
  `total == 0` for emulators that stub the syscall (show `n/a`).
- **There is no OS CPU-% API on PSL1GHT** — nothing in the headers. The frame work-time above is the
  honest processor-load proxy on a fixed-clock console.
