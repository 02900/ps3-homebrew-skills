---
name: psl1ght-build
description: >-
  Build & verify workflow for PS3 / PSL1GHT homebrew. Use when building, compiling, or
  deploying a PS3 homebrew project — the Dockerized toolchain (scripts/build.sh), output
  naming (src.self), keeping -Wall clean, working in small reversible increments, and the
  transient cc1/collect2 "internal compiler error: Segmentation fault" under emulation
  (not your bug — just retry).
---

# Build & verify workflow (PS3 / PSL1GHT)

> Rule of thumb that underlies most PS3 homebrew work: **you cannot run the build on the dev
> host** — only the user can, on a real PS3 / RPCS3. So favour code you can fully reason
> about, change in small reversible steps, build green in the toolchain image, and mark
> on-hardware behaviour as *unverified* until someone plays it.

- **Build through the Docker toolchain**, never assume a local PSL1GHT:
  ```bash
  DOCKER_DEFAULT_PLATFORM=linux/amd64 ./scripts/build.sh        # -> src.self
  ./scripts/build.sh pkg                                        # -> src.pkg (XMB install)
  PS3_IP=192.168.x.x ./scripts/deploy.sh                        # ps3load to a console
  ```
  `DOCKER_DEFAULT_PLATFORM` is read by the Docker CLI itself, so the scripts need no
  `--platform` flag (the image is x86_64; Apple Silicon emulates it). The toolchain image is
  `ghcr.io/02900/ps3-toolchain:latest`.
- **Output is named after the mount dir.** Mounted at `/src`, the Makefile's
  `TARGET := $(notdir $(CURDIR))` makes `src.elf` / `src.self` / `src.pkg`. Deploy/CI match by
  glob or `src.self`, not a repo-named file.
- **Keep `-Wall` clean.** The build uses `-Wall`; treat warnings as failures. Common one:
  `-Wmisleading-indentation` from `if (a) x; if (b) y;` on one line — split it.
- **Small, reversible increments.** Build green after every change; a wrong matrix / colour /
  sign often shows only as a black or garbled screen the user has to catch.
- **Transient toolchain segfaults are normal under emulation.** The PPU gcc 7.2 (`cc1` /
  `collect2`) sporadically crashes with "internal compiler error: Segmentation fault" when the
  x86_64 image runs emulated on Apple Silicon. It's not your code — just re-run.
  `scripts/build.sh` retries automatically; don't go hunting a code cause unless the failure is
  *deterministic* (same error every run).
