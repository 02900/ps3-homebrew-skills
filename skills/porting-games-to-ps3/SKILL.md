---
name: porting-games-to-ps3
description: >-
  Porting an existing game to PS3 homebrew (PSL1GHT). Use when bringing another game to PS3.
  Three strategies: (a) rewrite an engine game (Unity, etc.) in C with hand-rolled physics and
  explicit state machines; (b) wrap a C++/SFML/SDL/raylib game behind a thin framework-shim so
  its logic compiles nearly unchanged; (c) reuse a C++ engine's logic verbatim and rewrite only
  its I/O rind (window/input/render/asset+data load), baking any runtime data format (YAML/JSON)
  into C++ on the host. Covers C++/C interop, gcc-7.2/newlib gaps, STL on PSL1GHT, raylib name
  collisions, legacy-GL→raylib 2D, colour-key transparency, PCX→PNG, sprite sub-rects,
  virtual-resolution mapping, asset+data embedding, and the subtle traps each path hits.
---

# Porting a game to PS3 homebrew

Pick the strategy by what the original is:

- **Engine game (Unity, Godot, GameMaker…) → rewrite the logic in C/C++** against PS3 APIs
  (there's no engine to port). See **Strategy A**.
- **Already C++/C on a portable framework (SFML, SDL, raylib…) → wrap the framework behind a
  thin shim** so the game code compiles nearly unchanged. See **Strategy B**.
- **C++ engine wired directly to SDL2 + raw OpenGL + a data lib (yaml-cpp…), no single clean
  framework boundary → reuse the logic files verbatim and rewrite only the I/O rind.** See
  **Strategy C**.

Whatever the strategy, if the original loads config/level/character **data at runtime**, see
**Baking runtime data → C++** — it applies across all three.

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

---

## Strategy C — reuse the engine, rewrite only the I/O rind

When the original is C++ but wired **directly** to SDL2 + raw OpenGL + a data lib (yaml-cpp, etc.)
with no single framework namespace to shim, don't shim and don't rewrite the logic. **Split the
tree into pure logic vs the I/O rind, keep the logic files verbatim, and rewrite only the rind.**
(Case study: OpenFight, a 2D fighter — ~15 engine files reused unchanged, ~5 I/O files rewritten,
onto raylib/RSXGL.)

- **The logic is plain STL — it compiles unchanged.** State machines, animation timing, collision
  math, entity/object managers, move-sequence recognisers: bring them over as-is. Only these
  concerns are rewritten — **window+loop, input, rendering, asset load, data load**. Identify the
  handful of files that `#include <SDL2/…>` / GL / the data lib; those are the rind.
- **STL works on PSL1GHT** because RSXGL's libGL is C++, so the Makefile links with the C++ driver
  (`LD := $(CXX)`) → libstdc++ is present: `std::map/list/string/vector`, `shared_ptr`, `iostream`,
  **RTTI/`typeid`** and exceptions all link. You do *not* need `-fno-exceptions/-fno-rtti` if the
  game uses them (unlike the lean Strategy-B shim).
- **Type/compat shims let big files compile untouched.** Provide the numeric aliases the code
  expects instead of editing it: a `gl.h` shim (`typedef float GLfloat; typedef unsigned int
  GLuint;` — no GL) and a `compat.h` (`SDL_GetTicks()` → `(unsigned)(GetTime()*1000)` via raylib).
  Strip the `<SDL2/…>` includes from the headers you bring over (one `sed`), and drop the globals
  the rind owned to slim headers (e.g. a `graphicsCore.h` down to just the object registry).
- **Rewrite the data loader, not its consumers.** The one file that touched yaml-cpp became a
  `chardata.cpp` that builds the *same* runtime objects from baked structs (see below); Player,
  Animation, etc. never changed. Swap the leaf, keep the tree.

### raylib-on-PS3 I/O (rind rewrites)

- **⚠️ Identifier collisions with raylib's headers.** raylib's `KeyboardKey` enum defines
  `KEY_UP`/`KEY_A`/… (=265/65). A ported game with its own `enum { KEY_UP … KEY_A … }` collides
  ("redeclaration of 'KEY_A'") in any TU that includes both `raylib.h` and the game header. **Rename
  the game's enum** (e.g. `OFK_*`) across its files — mechanical `perl -pi -e 's/\bKEY_(…)\b/OFK_$1/g'`.
- **Legacy fixed-function GL → raylib 2D.** `glBegin`/`glVertex`/`gluPerspective`/`glTranslatef`
  don't exist. If the original draws a flat Z=0 scene through a *fixed* perspective camera,
  reproduce the **on-screen layout** in raylib screen space instead of a GL camera: at a fixed eye
  depth `d`, `gluPerspective(fovy)` maps a world half-height `d·tan(fovy/2)` onto `H/2`, so
  `scale = (H/2)/(d·tan(fovy/2))` px per world unit; `screen = center + (world − origin)·scale`
  (invert Y). Draw with `DrawTexturePro` — a **negative `source.width` flips horizontally**, which
  matches sprite facing-flip for free.
- **Colour-key transparency (alpha-less sprites).** Originals that key a colour to transparent (SDL
  surface mask passes) → at load: `ImageFormat(&img, PIXELFORMAT_UNCOMPRESSED_R8G8B8A8)` then
  `ImageColorReplace(&img, BLACK, BLANK)`, then draw with normal alpha blend. raylib-ps3 does have
  `ImageColorReplace`/`ImageFormat`/`LoadImageColors`.
- **⚠️ raylib can't decode PCX (and some other formats).** Convert to PNG on the host in the baker
  (Pillow: `Image.open(pcx).convert("RGB").save(png)`); embed the PNG.

### Runtime & mode traps (both C and C++ ports)

- **⚠️ Guard null derefs when running a reduced mode.** Two-player logic exercised with one player:
  a `opponent->index = …` write with `opponent == NULL` crashes as an **RPCS3 "Access violation
  writing location 0x44"** (null + member offset). Bring modes up incrementally and guard the
  absent side (`if (opponent) …`).
- **⚠️ Snapshot before iterating a container whose entries spawn mid-`update()`.** A projectile's
  `update()` can spawn a burst → adds to the object registry → invalidates a live map iterator.
  Take a `keys()` snapshot first, iterate that (skip already-removed, remove finished). Also **cull
  escaped projectiles** (off-stage past a bound) or missed shots leak forever on a 200 MB console.
- **Testing reality on RPCS3.** It **recompiles PPU modules on every new binary** (slow first boot —
  wait it out, ~60 s, it caches). Gamepad-driven behaviour can't be verified headless: a **clean
  boot still validates a lot** — malformed baked data or a bad texture crashes at *load/init* (esp.
  when the whole spawn-object graph is registered up front), so boot-clean proves data + init +
  decode; only live input/hit-trades need a real controller.

---

## Baking runtime data → C++ (any strategy)

Desktop games load config / levels / character data at runtime (yaml-cpp, JSON, XML…). PS3 has
**no runtime filesystem**, and cross-compiling a heavy C++ parser (yaml-cpp: exceptions, RTTI,
`std::string` churn) is a rabbit hole. **Bake the data into C++ on the host at build time** — it
kills both problems at once.

- **Host script (Python) parses the data → emits a committed C++ header of plain structs.** The PS3
  build needs no parser and no files. Keep the generated header + the embedded assets committed;
  the baker is a dev-time tool (document its deps — pyyaml, Pillow). Put the source data in a
  gitignored `vendor/` checkout of the original's assets.
- **Follow references to bake the whole graph.** If data points at more data (a character's move
  spawns a projectile defined in another file, which spawns a burst in a third), **BFS-discover**
  from the entry file and bake every reachable object into one table, keyed by the exact path
  strings the engine passes to its spawn/create calls — so the runtime lookup is a string match.
- **Replace the runtime loader, keep its consumers.** Rewrite only the file that parsed the format;
  have it build the same objects from the baked structs. Everything downstream is untouched.
- **⚠️ bin2o symbols from digit-leading filenames are invalid C identifiers.** `1punch00.png` →
  `1punch00_png` won't compile as an `extern`. Have the baker **sanitise** each asset to a C-safe
  name (prefix a `_`, replace non-alnum) and copy/convert the file to `data/<sanitised>.png`, then
  reference that symbol from the generated table. Dir-prefix the name (`ryu/idle00` → `ryu_idle00`)
  so frames from different objects don't collide.

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
