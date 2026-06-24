---
name: porting-games-to-ps3
description: >-
  Porting an existing game to PS3 homebrew (PSL1GHT). Use when bringing another game to PS3.
  Two strategies: (a) rewrite an engine game (Unity, etc.) in C with hand-rolled physics and
  explicit state machines; (b) wrap a C++/SFML/SDL/raylib game behind a thin framework-shim so
  its logic compiles nearly unchanged. Covers C++/C interop, gcc-7.2/newlib gaps, sprite
  sub-rects, virtual-resolution mapping, asset embedding, and the subtle traps each path hits.
---

# Porting a game to PS3 homebrew

Pick the strategy by what the original is:

- **Engine game (Unity, Godot, GameMaker…) → rewrite the logic in C/C++** against PS3 APIs
  (there's no engine to port). See **Strategy A**.
- **Already C++/C on a portable framework (SFML, SDL, raylib…) → wrap the framework behind a
  thin shim** so the game code compiles nearly unchanged. See **Strategy B**.

---

## Strategy A — rewriting an engine game

- **Read the original mechanic before assuming it.** One game's "health" was a *power
  tug-of-war* (a hit transfers power; you win by filling a single shared bar to 100), not a
  health-to-zero race. Porting it as the latter would be wrong. Read the source script, don't
  infer from the genre.
- **Map engine input to the pad explicitly.** Unity `CrossPlatformInput` axes/buttons → concrete
  DualShock buttons; write the mapping down (header comment + on-screen hint).
- **Replace `Rigidbody`/`Collider` with hand-rolled math.** Overlap = distance vs a radius, or
  AABB overlap; "raycast" = step + proximity test. PS3 has no physics engine to lean on.
- **State machines port cleanly** (round/match flow, AI FSM) — keep them explicit.
- **Record every deviation.** Where the port simplifies the original (e.g. a single damage base
  instead of an inner/outer split, or skipping knockback), say so in the commit and the roadmap
  so it's a known choice, not a silent bug.

---

## Strategy B — the C++/SFML framework-shim approach

When the original is **already C++ on a portable framework**, don't rewrite the game logic —
**wrap the framework behind a thin shim** backed by PS3 APIs, so the game code compiles nearly
unchanged.

- **One shim header per framework namespace.** Provide just the `sf::` types the game actually
  uses (grep the source — it's usually a small set): `sf::Texture` → `ya2d_Texture`, `sf::Sprite`
  → a textured-quad draw with position/scale/`IntRect`, `sf::RenderWindow` → tiny3d clear/flip,
  `sf::View` → a 2D camera offset applied to draws, `sf::Keyboard` → the pad mapping,
  `sf::Font`/`sf::Text` → `ttf_render` or Clay, `sf::Music`/`sf::Sound` → MikMod, plus value types
  (`sf::Vector2`, `sf::Color`, `sf::IntRect`, `sf::RectangleShape`). Implement behaviour
  incrementally — **stub first so it links**, then fill in rendering/input/audio.
- **Build against a stub shim first.** A header-only shim with the ~20 methods the game actually
  calls is enough to get the whole codebase *compiling & linking* (stub bodies), before any real
  rendering exists — a huge, verifiable milestone that de-risks the rest. Keep the texture handle
  an opaque `void*` in the header so it stays pure C++ (no ya2d include); the real backend `.cpp`
  fills it in later.

### Compiler / toolchain traps

- **C++↔C header interop.** The PS3 libs are C. Headers with their own
  `#ifdef __cplusplus extern "C"` guards (`tiny3d.h`, `io/pad.h`, `sysutil.h`) include normally;
  **un-guarded ones (`ya2d.h`) must be wrapped** in `extern "C"` from C++, or the C++ compiler
  mangles the names → undefined references. ⚠️ But wrapping a header that transitively pulls
  **system** headers (`ya2d.h` → `machine/malloc.h`) drags those into the block and clashes
  ("conflicting declaration … with 'C' linkage"). Fix: **pre-include the C system headers**
  (`<stdlib.h>`, `<malloc.h>`, `<string.h>`) *before* the `extern "C"` wrap so their include
  guards are already set. Give any C helper you vendor (e.g. `ttf_render.h`) its own `extern "C"`
  guard.
- **`ppu-g++` is gcc 7.2 → no C++20.** Build the source at `-std=gnu++17` (set `CXXFLAGS`
  separately from the C `-std=gnu99`; don't let `CXXFLAGS = CFLAGS` apply gnu99 to C++). Replace
  C++20-only constructs (concepts, ranges, `<bit>`) as you hit them. `-fno-exceptions -fno-rtti`
  keeps the binary lean if the game doesn't need them.
- **newlib's libstdc++ lacks `std::to_string` / `std::stoi` / `std::stof`** ("'to_string' is not
  a member of 'std'"). Provide them in a tiny `ps3_compat.h` (snprintf / strtol based) and
  **force-include it everywhere** via `CXXFLAGS += -include ps3_compat.h` — no edits to the
  original source needed.
- **⚠️ A slim shim won't pull the transitive includes the original leaned on.** The game called
  `abs()` on **floats**. Upstream, `<cmath>` arrived transitively (via SFML headers) so it bound
  to the float overload; the lean shim didn't include it, so `abs()` resolved to C's **`abs(int)`**
  and silently **truncated every float** — it wrecked AABB-overlap collisions and divided by zero
  in `flip /= abs(flip)`. No compile error; wrong results at runtime. Use `std::fabs` explicitly
  and `#include <cmath>` wherever the game does float math; never trust unqualified
  `abs`/`round`/`min`/`max` to resolve as they did upstream.
- **⚠️ A C header with non-`extern` globals breaks multi-TU C++ links.** `ya2d_controls.h` has
  `padData ya2d_paddata[7];` (no `extern`): in C that's a tentative def (merges); in **C++ it's a
  real definition**, so two C++ TUs including `<ya2d/ya2d.h>` → "multiple definition". Fix: include
  only the sub-headers you need (skip the offending one — use `io/pad.h` for input), e.g. via a
  private `ya2d_lite.h`.

### Rendering the shim

- **2D camera = subtract the view offset at draw time.** A side-scroller's `sf::View` becomes
  `screen = world - camera`; cull sprites outside the screen. No 3D pipeline to set up —
  ya2d/tiny3d quads in `tiny3d_Project2D`, all drawn at a constant `z` (its depth test is
  `LEQUAL`, so draw order = paint order — see the **psl1ght-rendering** skill).
- **⚠️ Map the game's virtual resolution onto tiny3d's fixed 848×512 canvas.**
  `tiny3d_Project2D()` gives a **fixed 848×512** 2D space whatever the output mode, but the game
  thinks in its own window size (e.g. 1920×1080, SFML Y-down). So the draw transform is two steps:
  `world → (apply sprite pos/origin/scale, then − camera) → · (848/winW, 512/winH)`. Bake the
  `canvas/window` factor into the vertex positions; don't fight it by resizing the window.
  `getSize()` must return the *virtual* size the game positions against (1920×1080), not 848×512,
  or the camera math (`width()/2` centering, `mapGridToPixels`) drifts.
- **⚠️ `ya2d_drawTextureEx` is full-image only — sub-rects need a raw tiny3d quad.** Sprite-sheet
  animation (`setTextureRect`) and texture atlases require partial UVs, which ya2d's helpers don't
  expose. Emit the quad yourself: `tiny3d_SetTextureWrap(0, t->textureOffset, t->textureWidth,
  t->textureHeight, t->rowBytes, (text_format)t->format, …, TEXTURE_NEAREST)` then
  `SetPolygon(TINY3D_QUADS)` + 4 × `VertexPos/Color/Texture`. **Normalize UVs by
  `textureWidth/Height`** (the allocated, possibly padded VRAM size), *not* `imageWidth`.
  `ya2d_Texture.format` already holds the tiny3d `text_format` enum (RGBA → `A8R8G8B8`).
  `TEXTURE_NEAREST` (=0) keeps pixel art crisp; building the corners from the full SFML transform
  `world = pos + (local − origin)·scale` makes negative `scale.x` (facing flip) fall out for free.
- **⚠️ Match SFML default-value semantics, or feedback loops oscillate.** A `RenderWindow`'s view
  defaults to the *centered* default view (center = size/2), not `(0,0)`. Side-scrollers often
  write the camera as `view.setCenter(x, height() - getView().getCenter().y)` — that's only stable
  because `height - height/2 == height/2` is a fixed point. If your `sf::View()` default-constructs
  center `(0,0)`, the term oscillates `height<->0` every frame and the whole scene jumps vertically
  (looked exactly like a GPU/buffer flicker — half the sprites cull out each frame, alternating).
  Initialize the window view to `getDefaultView()` in `create()`. Lesson: when a shim feeds its own
  getters back into setters, the *default* value must match the real framework's.
- **Cull off-screen sprites at draw time.** A wide level draws far more entities than fit on
  screen (e.g. ~210 tiles wide, ~30 visible). Compute the sprite's screen AABB and skip if it
  misses the 848×512 canvas — both a perf win and it keeps the per-frame tiny3d draw count in the
  range the renderer is proven at.

### Assets without a filesystem

- **Tilemaps / level configs** are usually plain `.txt`. Embed them with `bin2o` (name them
  `*.bin`) and parse from memory (`std::istringstream` over the embedded buffer — the game's
  `fin >> token` loops work unchanged), or ship them in the PKG `assets/`.
- **Asset paths → embedded buffers.** The game loads textures by file path; there's no filesystem,
  so map **path basename → bin2o buffer** in a small generated registry and have the shim's
  `loadFromFile` do the lookup → `ya2d_loadPNGfromBuffer`. No edits to the game's asset code.
- **⚠️ Bring the platform up lazily if the game loads assets before the window.** A game that
  loads every texture in its ctor *before* `window.create()` will crash on PS3 — it's `create()`
  that inits tiny3d/ya2d, so the first `loadFromFile` decodes a PNG into an uninitialised RSX. Fix:
  make platform bring-up **idempotent + lazy** — call it from both `create()` *and* the first
  `loadFromFile()`, guarded by a `ready` flag — so whichever the game reaches first wins.
