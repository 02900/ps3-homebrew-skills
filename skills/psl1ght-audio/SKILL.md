---
name: psl1ght-audio
description: >-
  Audio on PS3 via MikMod. Use when adding music or sound effects to PS3 homebrew — MikMod plays
  tracker modules / PCM samples (NOT OGG/MP3); synthesize SFX in code or load samples from
  memory via an MREADER; hand-built WAVs must be little-endian (the PPU is big-endian); loop long
  music as a sample; and init defensively because a bad audio init can hang the console.
---

# Audio (MikMod)

MikMod plays tracker **modules** and **samples** (PCM) — it **cannot decode OGG/MP3**. Drive it
with `MikMod_Update()` once per frame and reserve voices with `MikMod_SetNumVoices(music, sfx)`
(sample playback needs sfx voices reserved).

> **raylib-on-PS3 has no working audio device.** On the RSXGL/raylib stack, raylib's
> `InitAudioDevice`/`LoadSound`/`PlaySound` aren't linkable (undefined references — even from
> dead code). MikMod, below, *is* the audio path for raylib ports too: embed the game's `.wav`s
> and load them with `Sample_LoadGeneric` via an MREADER (see *Load audio from memory*).

## Synthesize SFX in code — no assets needed

You can generate sound effects procedurally: build a PCM waveform with math (a decaying sine for
a "thud", a falling sweep for a "pew", noise for an explosion), wrap it as a little-endian WAV in
memory, and load it as a MikMod sample. Self-contained (no asset files), great for prototyping,
placeholders, or a stylized retro feel.

```c
#define SR 22050
short pcm[SR/2];
int n = SR * 8 / 100;                       /* 80 ms */
for (int i = 0; i < n; i++) {
    float t = (float)i / SR, env = expf(-t * 22.0f);     /* fast decay */
    float s = sinf(2*M_PI*150.0f*t) * 0.7f + noisef() * 0.3f;
    pcm[i] = (short)(env * s * 22000.0f);
}
/* -> make_wav(pcm, n) into a byte buffer, then Sample_LoadGeneric (see below) */
```

`noisef()` is a tiny LCG mapped to −1..1. Vary frequency/decay/noise mix per effect. (Note: if you
want to avoid the `libm` dependency for `sinf`/`expf`, a square wave via a manual phase wrap +
linear envelope sounds fine and needs no math lib.)

## Load audio from memory via an MREADER

MikMod's loaders (`Sample_LoadGeneric`, `Player_LoadGeneric`) take an **`MREADER`**, not a raw
pointer. Implement a ~5-function in-memory reader over an embedded buffer (from `bin2o`) and pass
`&reader.core`:

```c
typedef struct { MREADER core; const unsigned char *data; long size, pos; } MemReader;
/* Get -> byte or EOF; Read -> 1 on FULL read else 0; Seek -> 0 on success (non-zero
 * error); Tell -> pos; Eof -> pos>=size.  NOTE the inverted Read vs Seek conventions. */
SAMPLE *s = Sample_LoadGeneric(&mr.core);   /* NULL on failure */
```

Embed the `.wav` with `bin2o` by naming the file `*.bin` in `data/` (the Makefile's `bin2o` rule
keys on extension; MikMod parses the RIFF header, not the filename).

Convert real assets to a clean canonical WAV first (strips odd chunks that loaders trip on), e.g.
with ffmpeg:
```bash
ffmpeg -i in.ogg -ac 1 -ar 22050 -c:a pcm_s16le -map_metadata -1 -f wav data/music.bin
```

## Hand-built WAV bytes must be little-endian (PPU is big-endian) ⚠️

WAV is little-endian by spec. If you **build** a WAV yourself (e.g. for synthesized SFX), write
the header fields *and* the int16 PCM as explicit LE bytes — a naive `u32`/`s16` store is
big-endian on the PPU and MikMod will misread it. (Real `.wav` files from ffmpeg are already LE,
so this only bites hand-built buffers.)

```c
static void put_u16le(unsigned char *b, unsigned v){ b[0]=v; b[1]=v>>8; }
static void put_u32le(unsigned char *b, unsigned v){ b[0]=v; b[1]=v>>8; b[2]=v>>16; b[3]=v>>24; }
```

## Loop long music as a sample (the OGG/MP3 way)

Since MikMod can't play OGG/MP3 and a module would mean re-authoring, decode the track to PCM WAV
on the host (ffmpeg) — or synthesize a melody — and play it as **one long looping sample**: set
`SF_LOOP` + `loopend`, and play with `SFX_CRITICAL` so transient SFX voices can't steal it.

```c
music->flags |= SF_LOOP; music->loopstart = 0; music->loopend = music->length;
SBYTE v = Sample_Play(music, 0, SFX_CRITICAL);
if (v >= 0) Voice_SetVolume(v, 150);        /* duck under SFX */
```

## Init audio defensively — it can hang the console ⚠️

A bad audio init can **freeze a PS3** (double-init is a known hang), and you can't hear it on the
dev host. Guard every step; on any failure set an `audio_ok = 0` flag and make all calls no-ops
(degrade to silence). Isolate audio in its own file so it's easy to disable.
