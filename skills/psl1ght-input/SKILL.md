---
name: psl1ght-input
description: >-
  Controller input on PS3 via the PSL1GHT pad API. Use when reading DualShock/gamepad input in
  PS3 homebrew — the padData.len==0 "retain the last packet" gotcha (and the one-reader-per-port
  rule), per-port reads for two players, edge-trigger vs level-read actions, the analog
  (0,0)=no-data deadzone guard, and the RSXGL/OpenGL gotchas (init the pad AFTER the GPU context;
  swap-pacing or input lag grows to seconds) that raylib hides.
---

# Input — DualShock via the PSL1GHT pad API

## `padData.len == 0` means "no new data" — retain the last packet ⚠️

This bit a project **three times** (phantom flailing, false "disconnected", a charge stalling
while held). `ioPadGetData` / `cellPadGetData` sets `len = 0` on frames with no input change
(e.g. a button held while the sticks are still). If you zero the struct every frame and use it
as-is, held buttons read as released on those frames.

**Do:** read into a temp; refresh a retained per-port copy only when `len > 0`; reuse it
otherwise so held inputs stay held.

```c
static padData held[2];                 /* persists across frames */
padData pd[2]; int conn[2] = {0,0};
ioPadGetInfo(&pad_info);
for (int i = 0; i < 2; i++) {
    padData tmp; memset(&tmp, 0, sizeof tmp);          /* zero-init: no stack garbage */
    if (pad_info.status[i] && ioPadGetData(i, &tmp) == 0) {
        conn[i] = 1;
        if (tmp.len > 0) held[i] = tmp;                /* fresh data: remember it     */
        pd[i] = held[i];                               /* else reuse last known state */
    } else {
        conn[i] = 0; memset(&held[i], 0, sizeof held[i]);  /* forget on disconnect    */
    }
}
```

**Don't:** gate *connection* on `len > 0` (a motionless pad would read as disconnected), and
**don't** read into an uninitialized struct (a phantom/unconfigured port — common on RPCS3 —
leaves stack garbage that sends a character flailing into a corner).

⚠️ **Only ONE reader per port per frame.** `ioPadGetData` returns the fresh packet (`len > 0`)
to the *first* caller in a frame; a second call the same frame gets `len == 0`. So if two
places read port 0 (e.g. a `start_pressed()` helper *and* an event-poll backend), the second is
permanently starved — it never sees `len > 0`, so its retained state never initializes and it
produces no input (symptom: "only Start works, nothing else"). Have a single pad-read site and
share its state.

## Two controllers = two ports

`padInfo.status[i]` flags each connected port; `ioPadGetData(i, &data)` reads a specific one.
Port 0 = player 1, port 1 = player 2 — same layout each. Accept menu/quit buttons from *either*
pad. On RPCS3, mapping the *same* physical pad to two players makes both characters move
together; that's expected, not a bug.

## Edge-trigger actions, level-read holds

One-shot actions (fire, melee, jump, menu select) must be edge-detected so a held button does
not repeat; continuous actions (charge, move) read the current level.

```c
u32 cur = (pd.BTN_SQUARE?1:0) | (pd.BTN_CIRCLE?2:0);
u32 pressed = cur & ~prev;   prev = cur;   /* pressed = this frame's new presses */
```

## Analog deadzone + the "(0,0) = no data" guard

Map a stick byte (0..255, centre 128) through a deadzone. Treat **both axes exactly 0** as "no
analog data this frame" (digital pad / not read), not full down-left — otherwise the character
drifts on startup.

```c
if (!(pd.ANA_L_H == 0 && pd.ANA_L_V == 0)) { mx = axis(pd.ANA_L_H); mz = -axis(pd.ANA_L_V); }
```

## On RSXGL / OpenGL: two input failures that don't happen with Tiny3D ⚠️

The raw RSXGL/OpenGL path (e.g. `ps3-gl-test`) hits two pad problems that Tiny3D/ya2d apps
never see — both because RSXGL owns the GPU init that Tiny3D otherwise did for you.

### 1. Init the pad AFTER the GPU context

RSXGL's `eglInitialize` / `eglMakeCurrent` **resets the lv2 IO subsystem**. A pad initialised
*before* it is dead: `padInfo.status[0]` never sets, so nothing reads. The trap is the symptom —
the app *seems* to respond only to **"Start closes it"**, but that close is the system
`SYSUTIL_EXIT_GAME` event delivered via `sysUtilCheckCallback`, **not** your `BTN_START`. So
"only Start works" here means the pad is entirely dead, the opposite of the one-reader-per-port
case above. (Tiny3D/ya2d don't hit this — they init the pad after their video init.)

```c
// ... eglMakeCurrent(dpy, surface, surface, ctx);   // GPU context first
sysModuleLoad(SYSMODULE_IO);                          // THEN the pad
ioPadInit(7);
```

**Diagnosing loop-alive vs pad-dead** (do this before guessing): add a throwaway per-frame
animation, e.g. auto-rotate the camera each frame regardless of input. If it animates, the
render loop is fine and *only* the pad is dead → it's this init-order bug. If it's frozen, the
loop itself is stuck (see #2).

### 2. `eglSwapBuffers` doesn't throttle → "input lag" that grows to seconds

RSXGL's `eglSwapBuffers` does **not** sync to the display. An unthrottled loop (e.g. `usleep(1000)`
≈ 1000 fps) submits frames far faster than the console shows them; the backlog turns into
*seconds* of growing input-to-screen latency — it looks exactly like the pad responding ~20s
late, but the pad is fine. **Pace the loop to ~60 fps yourself** (`gettimeofday` + sleep the
remainder of a 16.6 ms budget). Note: `eglSwapInterval` is declared in RSXGL's `EGL/egl.h` but
**not implemented** in `libEGL` → linking it is an undefined-reference error; don't use it.

```c
struct timeval prev; gettimeofday(&prev, NULL);
while (running) {
    /* ... draw ... */  eglSwapBuffers(dpy, surface);  sysUtilCheckCallback();
    struct timeval now; gettimeofday(&now, NULL);
    long us = (now.tv_sec - prev.tv_sec)*1000000L + (now.tv_usec - prev.tv_usec);
    if (us < 16666L) usleep(16666L - us);
    gettimeofday(&prev, NULL);
}
```

### raylib hides both

The `raylib-ps3` port (e.g. `ps3-raylib-test`) sets up EGL, inits the pad in the right order,
and paces frames (`SetTargetFPS(60)`) for you. Use raylib's gamepad API and none of the above
applies: `IsGamepadButtonPressed(0, GAMEPAD_BUTTON_LEFT_FACE_UP/DOWN/LEFT/RIGHT)` for the D-pad
(edge-triggered already), `GAMEPAD_BUTTON_MIDDLE_RIGHT` for Start, and
`GetGamepadAxisMovement(0, GAMEPAD_AXIS_LEFT_X/Y)` (already normalised −1..1) for the sticks.
