---
name: psl1ght-rendering
description: >-
  2D/3D rendering on PS3 with Tiny3D + ya2d, and Clay for UI. Use when drawing sprites, shapes,
  text, a camera, or any HUD/menu in PS3 homebrew — the ARGB-vs-RGBA colour-format trap, a
  deterministic software camera, the 2D depth test (LEQUAL → draw order = paint order, don't
  bias z per-sprite), building ALL UI/HUD in Clay to avoid render glitches, and /dev_flash
  system fonts.
---

# Rendering (Tiny3D + ya2d + Clay)

## Colour format mismatch ⚠️

- `tiny3d_Clear()` takes **ARGB** (`0xAARRGGBB`).
- `ya2d_*` fills and `tiny3d_VertexColor()` take **RGBA** (`0xRRGGBBAA`).

Mixing them silently produces wrong colours (e.g. a "blue" clear coming out red). Keep clear
constants separate and comment the byte order at the define.

## Prefer a deterministic software camera when you can't iterate on-device

Tiny3D has a full matrix pipeline (`tiny3d_Project3D`, `SetProjectionMatrix`,
`SetMatrixModelView`, `matrix.h`), but its multiply order / FOV conventions are easy to get
subtly wrong, and a wrong camera = a black screen you can only debug on hardware. A small,
fully-understood **pinhole projection** (lookAt basis + perspective divide) feeding Tiny3D's 2D
primitives is predictable and reviewable:

```c
/* basis: f = norm(target-eye); r = norm(cross(up,f)); u = cross(f,r) */
float d[3]; v_sub(p, eye, d);
float vz = v_dot(d, f); if (vz < 0.05f) return 0;        /* behind camera */
*sx = CX + FOCAL * v_dot(d, r) / vz;
*sy = CY - FOCAL * v_dot(d, u) / vz;
*scale = FOCAL / vz;                                     /* px per world unit at depth */
```

General principle: **when feedback loops are slow/expensive, choose the approach you can verify
by reading, not by running.**

## 2D draw order = paint order (via the depth test), not "no z-buffer"

For a **3D** scene, draw far-to-near: sort objects by camera-forward depth
(`dot(pos - eye, forward)`), draw the larger-depth one first. Fine for a handful of actors.

> **2D fact (verified by disassembling `libtiny3d`):** `tiny3d_Project2D` is *not* z-less — it
> enables the depth test with func **`LEQUAL`**, depth-write **on**, and clears depth to
> **far**. So overlapping 2D quads compose in **draw order for free** *as long as they all use
> one constant `z`* (a later draw at the same `z` passes `≤` and overwrites). ⚠️ The trap: don't
> bias `z` per-sprite to "fix" ordering — a later, *larger* `z` then **fails** `≤` and the
> sprite vanishes wherever it overlaps an earlier one. Draw back-to-front at a single `z` and let
> LEQUAL do the rest.

## Clamp the footprint edge, not the centre

Clamp `pos ± half-extent` to the bounds so the body stops at the wall, not its centre. A
camera-facing **billboard is flat in Z**, so its Z footprint is 0 (it can reach the front/back
edge); give X a real half-width. Revisit when real meshes replace billboards.

## Prefer Clay for ALL 2D UI/HUD — it avoids render glitches ⚠️

**Build every HUD / menu / overlay with the Clay layout engine (`extern/clay-ps3`), not
hand-drawn `ya2d`/`ttf` calls.** Reserve raw `ya2d`/`tiny3d` primitives for the scene
(floor, sprites, projectiles) only.

**Why (hard-won):** mixing hand-drawn `display_ttf_string` / `ya2d_drawRectZ` HUD draws with the
scene produced **transient single-frame render glitches** — warped bars, scattered text, stray
geometry across the frame — that were maddening to chase (a `LINES` → quads attempt didn't fix
it). The glitches **disappeared the moment the HUD was moved into Clay** (`clay_render` issues
its draws through one consistent path). So a split HUD (some Clay, some hand-drawn) is the worst
case; go all-Clay.

Two patterns cover almost any game HUD purely in Clay:
- **Progress bar** = a fixed-size track with a `CLAY_SIZING_PERCENT(frac)` fill child + a
  `CLAY_SIZING_GROW(0)` remainder (e.g. a health / balance / energy bar).
- **Tick marks / centered markers** = `CLAY_FLOATING` children
  (`attachTo = CLAY_ATTACH_TO_PARENT`, `.attachPoints`, `.offset = {px,0}`) so they overlay a bar
  without disturbing its fill layout.

Text, borders, images and semi-transparent backgrounds all render (alpha blends).

**Build the Clay UI as a C TU even from a C++ game.** The `CLAY(...)` / `CLAY_TEXT_CONFIG(...)`
macros are C compound-literals — happiest under `-std=gnu99` (the renderer's own language) and
finicky in C++. Keep each screen's layout in a `.c` file and call it from the C++ game through a
tiny `extern "C"` bridge (`clay_render_menu(title, items, n, sel)`), passing plain C
strings/ints. Keep `CLAY_IMPLEMENTATION` in exactly one TU (the renderer).

⚠️ **`CLAY_IDI(label, i)` / `CLAY_ID(label)` need a string *literal*** — the macro stringifies
it, so a ternary (`CLAY_IDI(cond ? "A" : "B", 0)`) fails to compile. Use a fixed literal label
and disambiguate with the index: `CLAY_IDI("Tick", side*3 + k)`. (Note: the PS3 Clay renderer
ignores `cornerRadius` — borders are square.)

## Fonts come from `/dev_flash`

System TTFs (`/dev_flash/data/font/SCE-PS3-*.TTF`) are present on real consoles and RPCS3. Load
them via the `ttf_render` helper; don't ship your own for basic UI.
