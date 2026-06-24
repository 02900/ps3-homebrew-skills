# PS3 Homebrew Skills

Hard-won conventions for porting games to **PS3 homebrew (PSL1GHT)**, accumulated across ports
and packaged as **[Claude Code](https://code.claude.com) skills** so they're applied
automatically while you work — and as a shared git submodule so every project stays in sync
instead of each carrying its own drifting copy.

Each skill is **what to do**, **why**, and the **trap it avoids**.

> Rule of thumb that underlies most of this: **you usually cannot run a PS3 build on the dev
> host** — only on a real PS3 / RPCS3. So favour code you can fully reason about, change in small
> reversible steps, build green in the toolchain image, and mark on-hardware behaviour as
> *unverified* until someone plays it.

## Skills

This repo is a Claude Code **skills-directory plugin** (`ps3-homebrew`). Each skill auto-loads
when relevant, or you can invoke it explicitly with `/ps3-homebrew:<name>`:

| Skill | Covers |
| --- | --- |
| [`psl1ght-build`](skills/psl1ght-build/SKILL.md) | Dockerized toolchain, `src.self`/`src.pkg` output, `-Wall`, small increments, transient emulation segfaults |
| [`psl1ght-input`](skills/psl1ght-input/SKILL.md) | DualShock pad: `len==0` retained-packet gotcha, one-reader-per-port, two ports, edge vs level, deadzone |
| [`psl1ght-rendering`](skills/psl1ght-rendering/SKILL.md) | Tiny3D + ya2d colour formats, deterministic camera, 2D depth (`LEQUAL`) draw order, **Clay for all UI**, `/dev_flash` fonts |
| [`psl1ght-game-loop`](skills/psl1ght-game-loop/SKILL.md) | XMB-exit handshake, fixed timestep, fixed object pools, `bin2o` asset embedding |
| [`psl1ght-audio`](skills/psl1ght-audio/SKILL.md) | MikMod: synthesize/load samples from memory, little-endian WAVs, looping music, defensive init |
| [`porting-games-to-ps3`](skills/porting-games-to-ps3/SKILL.md) | Two porting strategies: rewrite an engine game (Unity…) in C, or wrap a C++/SFML game behind a thin shim |

## Use it in a project (git submodule)

Add it under `.claude/skills/` so Claude Code discovers it (mirrors how the Clay UI lib is
vendored at `extern/clay-ps3`):

```bash
git submodule add ../ps3-homebrew-skills.git .claude/skills/ps3-homebrew
git commit -am "skills: vendor ps3-homebrew patterns as a submodule"
```

The relative URL resolves to the same GitHub org as the superproject. On a fresh clone:

```bash
git clone --recurse-submodules <your-repo>          # or, after a plain clone:
git submodule update --init                          # fetch/refresh the skills
```

Claude Code auto-discovers it as the plugin `ps3-homebrew@skills-dir` **on the next session**
(no marketplace, no install step). To use the skills in *every* project (not just ones with the
submodule), clone this repo into `~/.claude/skills/ps3-homebrew/` instead.

### Keeping projects in sync

This repo is the **single source of truth**. To pull the latest patterns into a project:

```bash
git submodule update --remote .claude/skills/ps3-homebrew
git commit -am "skills: bump ps3-homebrew"
```

## TL;DR checklist for a new input/render feature

1. Read input from the **retained** pad packet, never assume fresh data each frame.
2. **Edge-detect** one-shot actions; level-read holds.
3. Watch the **colour format** (clear = ARGB, ya2d/vertex = RGBA).
4. Build **all 2D UI/HUD in Clay** (not hand-drawn ya2d/ttf) — a split HUD causes transient
   render glitches; keep raw primitives for the scene only.
5. Keep the **camera math deterministic**; in 2D draw at a constant `z` (depth test is `LEQUAL`).
6. Build **green under `-Wall`** in the toolchain image; mark on-hardware as unverified.
7. **Guard audio init defensively** — a bad init can hang the console, and you can't hear it on
   the dev host.
8. **Document deviations** from the source game.

## Provenance

Distilled from two PS3 ports in the `02900` org:
[ki-blast-arena](https://github.com/02900/ki-blast-arena) (a 3D C fighter, Unity → rewrite) and
[ps3-mega-mario](https://github.com/02900/ps3-mega-mario) (a 2D C++/SFML platformer →
framework-shim).

## License

MIT.
