---
name: psl1ght-game-loop
description: >-
  Game loop & simulation foundations for PS3 homebrew. Use when setting up the main loop,
  exiting cleanly to the XMB, timing the simulation, managing transient objects, or embedding
  assets — the per-frame sysUtilCheckCallback XMB-exit handshake, fixed timestep, fixed object
  pools, and bin2o asset embedding.
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
