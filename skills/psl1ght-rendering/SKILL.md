---
name: psl1ght-rendering
description: >-
  2D/3D rendering on PS3 with Tiny3D + ya2d, and Clay for UI. Use when drawing sprites, shapes,
  text, a camera, or any HUD/menu in PS3 homebrew — the ARGB-vs-RGBA colour-format trap, a
  deterministic software camera, the 2D depth test (LEQUAL → draw order = paint order, don't
  bias z per-sprite), building ALL UI/HUD in Clay to avoid render glitches, and /dev_flash
  system fonts. Also covers the raw RSXGL/OpenGL path (ps3-boilerplate-rsxgl, mega-mario rsxgl-backend):
  per-pixel __builtin_bswap32 for texture colours, fragment-shader uniforms being ignored, and
  batching quads (+ per-string text flush) to hit 60fps without RSXGL text corruption.
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

⚠️ **Enable alpha blending BEFORE `clay_render`, or every TTF glyph draws on an opaque
BLACK BOX.** The glyph quads are `A4R4G4B4` textures whose non-glyph pixels are transparent;
with blending off they composite as solid black, so each text string sits in an ugly black
rectangle. Set it once per frame right after `tiny3d_Clear`, before `tiny3d_Project2D()` /
`reset_ttf_frame()` / `clay_render()` (the exact setup `ps3-homebrew-showcase` uses):
```c
tiny3d_Clear(clear, TINY3D_CLEAR_ALL);
tiny3d_AlphaTest(1, 0x10, TINY3D_ALPHA_FUNC_GEQUAL);
tiny3d_BlendFunc(1, TINY3D_BLEND_FUNC_SRC_RGB_SRC_ALPHA | TINY3D_BLEND_FUNC_SRC_ALPHA_SRC_ALPHA,
                    TINY3D_BLEND_FUNC_DST_RGB_ONE_MINUS_SRC_ALPHA | TINY3D_BLEND_FUNC_DST_ALPHA_ZERO,
                    TINY3D_BLEND_RGB_FUNC_ADD | TINY3D_BLEND_ALPHA_FUNC_ADD);
tiny3d_Project2D();
```
Put this in a C helper (the `... | ...` enum-OR needs an implicit int→enum that C allows but
C++ rejects without casts), and call it from the C++ game.

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
them via the `ttf_render` helper; don't ship your own for basic UI. **`SCE-PS3-RD-R-LATIN.TTF`
(Rodin, the XMB UI font) is the clean, legible default** — nicer than the `VR` face.

⚠️ **`ttf_render`'s glyph atlas slots are 32×32, so keep Clay `fontSize` ≤ ~30.** A glyph is
rasterized into a 32×32 cell then the quad is scaled to the requested size; anything bigger is
upscaled and looks **blocky/pixelated** (a `fontSize: 64` title is mush). For a crisp large
title either cap at ~30 or bump the atlas cell size in `ttf_render.c` (and the
`tiny3d_AllocTexture` size) — the default 32² caps ~30px.

---

# Raw RSXGL (OpenGL) path — parallel to Tiny3D

Some ports render 2D **directly on RSXGL** (OpenGL 3.1 over the RSX: EGL context, GLSL, VBOs)
instead of Tiny3D — `ps3-boilerplate-rsxgl`, and mega-mario's `rsxgl-backend`. It's standard
column-major GL (none of Tiny3D's row-vector/+Z quirks), but the RSX driver has its own traps.
See the **input** skill for the two load-bearing app gotchas (init the pad AFTER the EGL
context; `eglSwapBuffers` doesn't throttle → pace to 60fps yourself) and link with the **C++
driver** (`LD:=$(CXX)`, RSXGL's libGL is C++).

## Textures come out channel-rotated — byte-swap every pixel ⚠️

Upload `GL_RGBA`/`GL_UNSIGNED_BYTE` bytes `[R,G,B,A]` and the sampler returns them **rotated**
(orange → magenta, green → purple). The tell: the `glClearColor` background looks right (it's
not a texture) while every sprite is mis-coloured. Fix on the CPU — `__builtin_bswap32` each
32-bit pixel (`[R,G,B,A] → [A,B,G,R]`) before `glTexImage2D`:

```c
uint32_t *px = (uint32_t *)rgba;
for (size_t i = 0, n = (size_t)w * h; i < n; i++) px[i] = __builtin_bswap32(px[i]);
```

Known "buggy PS3 opengl driver" quirk — **raylib-ps3 does the identical thing** (`rtextures.c`
`LoadTextureFromImage`, comment "Handle buggy PS3 opengl driver and swap endianess"), which is
why raylib gets correct colours from the same `glTexImage2D(GL_RGBA)` call. White/symmetric
textures (font atlas, 1×1 white) are unaffected either way. (This is the RSXGL cousin of the
Tiny3D ARGB-vs-RGBA trap above — colour byte order bites on both paths, differently.)

## RSXGL ignores extra fragment-shader uniforms ⚠️

Only the texture **sampler** and **vertex** uniforms (the MVP/ortho matrix) actually work. Any
other uniform read only in the fragment shader — an `int` mode, a `mat4` — comes back dead:
`glGetUniformLocation` returns **-1** and the value never reaches the shader (reads as 0; a zero
`mat4` → black/blank screen). Fragment dynamic branching on a uniform is likewise not honoured
by the Cg compiler. **Do per-fragment variation on the CPU** (bake it into the texture or vertex
data), not via a fragment uniform. (Cost two dead ends — a uniform swizzle selector, then a mat4
permutation — before the CPU byte-swap above.)

## Batch quads to hit 60fps — but flush text per-string ⚠️

One `glBufferData`+`glDrawArrays` per quad is hundreds of tiny uploads/frame — a debug grid of
~500 line-quads dropped mega-mario to ~20fps. **Batch:** accumulate quads into a CPU buffer,
flush only on **texture change / buffer full / frame end**. Flushing on every texture change
preserves painter's draw order, and consecutive same-texture quads (grid lines, solid rects)
coalesce to ~1 draw → back to 60fps. **Exception — flush text per-string** so each glyph run
stays a small draw: a big merged single-font-atlas `glDrawElements` is exactly what corrupts
glyph vertices on RSXGL (the same failure raylib's rlgl hits when it merges all text into one
batch). Controlling the batching yourself — small text draws, no merge — is the whole reason to
go raw instead of raylib.

A built-in **8×8 bitmap font** baked into an atlas texture (public-domain `font8x8`) covers ASCII
with no TTF/FreeType and no `/dev_flash` dependency — a good fit for the raw path.
