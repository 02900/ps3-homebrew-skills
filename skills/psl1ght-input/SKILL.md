---
name: psl1ght-input
description: >-
  Controller input on PS3 via the PSL1GHT pad API. Use when reading DualShock/gamepad input in
  PS3 homebrew — the padData.len==0 "retain the last packet" gotcha (and the one-reader-per-port
  rule), per-port reads for two players, edge-trigger vs level-read actions, and the analog
  (0,0)=no-data deadzone guard.
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
