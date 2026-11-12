# pes13-dsound

Experimental `dsound.dll` fixes and diagnostics for **missing short sound effects** in the PES 2013 Switch port (FEXTendo / Wine-NX).

> Unofficial project. Not affiliated with FEXTendo, Konami, or CRI ADX.  
> This is a **partial fix**, not a final solution. Read “Current status” first.

**Language:** [中文](README.md) | [English](README.en.md)

---

## Current status (2026-11-12)

| Sound | Original baseline | With this repo's patches |
|---|---|---|
| BGM / commentary / crowd | Works | Works |
| Referee whistle / kicks / ball touches | **Silent** | Partially restored, still unstable |
| Menu UI clicks | **Silent** | First two clicks audible, third onward silent |

Conclusions:

- The original problem is **all short 6-channel effects** (whistle, kicks, menu UI), not just kicks.
- The patches restore some of them, but **ADX voice reuse / looping behaviour** still causes the third and later short effects to disappear.
- This is not caused by the volume slider, `settings.dat`, or the native Windows `dsound.dll`.

---

## Symptoms

- The same port package has complete audio on PC (native Windows DirectSound).
- On Switch:
  - BGM, commentary, and crowd work;
  - short effects such as referee whistle, kicks, ball touches, and menu UI clicks are silent;
  - setting “match sound effects volume” to maximum has no effect;
  - replacing `dsound.dll` with the native Windows one hangs before the game starts.
- A debug `dsound.dll` shows that the 6-channel buffers **do contain real PCM**, but their volume is set very low:

```
fmt tag=0xFFFE ch=6 rate=48000 bits=16
buflen=49152  (~85 ms)
flags=0x000580A0
vol=-2570  or  vol=-10000
```

`0x000580A0` decodes to:

```
DSBCAPS_GLOBALFOCUS | DSBCAPS_GETCURRENTPOSITION2 |
DSBCAPS_LOCDEFER | DSBCAPS_CTRLVOLUME | DSBCAPS_CTRLFREQUENCY
```

There is no `DSBCAPS_CTRL3D` and no `DSBCAPS_LOCHARDWARE`.  
So this is neither 3D positioning nor a failed hardware-voice request; it is a **normal 6-channel short buffer with very low volume / looping state**.

---

## Test environment

- Switch (Atmosphere / homebrew enabled)
- FEXTendo `0.4.2`
- Wine-NX `5.3.1.904` (`autorunhq/autorun` @ `1bc4e45163f0d2328cdfd35c7f471dd9821bb879`)
- FEX-Emu (x86 → ARM64)
- Box64 `dae0917c47b4edd8956f314210417a20fd225c4b`
- Game: PES 2013 Chinese version
- Original `dsound.dll`:
  - size `1,200,128` bytes
  - SHA256 `6258E908BC589AD916268655B4892840900DD7584A4FDDF8AE36F9E91FFE46A6`

---

## Repository layout

```
.github/workflows/build-dsound.yml   GitHub Actions build
patches/
  dsound-fix.patch                  primary flags + LOCHARDWARE fallback
  dsound-lfe.patch                  fold 5.1/7.1 LFE into L/R
  dsound-stereo.patch               force the device to 2 channels
  dsound-dump.patch                 diagnostics + short 6ch volume/one-shot (combined)
  dsound-probe.patch                logging build: writes dsound-probe.log
  dsound-hwcaps.patch               non-logging build: caps only
```

---

## Build

1. Fork / upload this repository to GitHub.
2. Actions → `build-dsound` → **Run workflow**.
3. Inputs:
   - `logging: true` — writes `C:\dsound-probe.log`, useful for diagnostics.
   - `logging: false` — caps changes only, less logging.
4. Download the artifacts:
   - `dsound-dll`
   - `dsound-source`

Only `dsound.dll` is built; Mesa/DXVK are not rebuilt. A run usually finishes in about 2 minutes.

---

## Patch details

### `dsound-fix.patch`

- Merges capability flags (`CTRL3D` / `CTRLVOLUME` etc.) when an existing primary buffer is returned.
- For non-primary `DSBCAPS_LOCHARDWARE`, no longer returns `DSERR_UNSUPPORTED`; falls back to a software buffer.

> Early hypothesis. Logs show the game never requests `LOCHARDWARE` or `CTRL3D`, so these are not the root cause. Kept because they are harmless.

### `dsound-lfe.patch`

Folds the previously discarded LFE channel into L/R in both `put_surround512stereo` and `put_surround712stereo`:

```c
case 3:
    value *= 0.7f;
    dsb->put_aux(dsb, pos, 0, value);
    dsb->put_aux(dsb, pos, 1, value);
    break;
```

### `dsound-stereo.patch`

Clamps the device format to 2 channels at the end of `DSOUND_WaveFormat()`:

```c
if (w->nChannels > 2) {
    w->nChannels = 2;
    w->nBlockAlign = ...;
    w->nAvgBytesPerSec = ...;
    w->dwChannelMask = SPEAKER_FRONT_LEFT | SPEAKER_FRONT_RIGHT;
}
```

This makes 6ch/8ch secondary buffers go through Wine's downmix path instead of feeding 5.1 data directly to the 2-channel audout backend.

### `dsound-dump.patch` (current combined version)

Four parts:

1. **Per-voice snapshots**
   - The first time each 6ch buffer has meaningful data, writes:
     ```
     drive_c\dsound-voice-01.raw
     drive_c\dsound-voice-01.txt
     ```
   - txt contents:
     ```
     seq peak vol pan flags ch0..ch5
     ```

2. **2-second final-mix recording**
   - Triggered when a 6ch write has peak > 3000.
   - Records the final `DSOUND_PerformMix` output for 2 seconds:
     ```
     drive_c\dsound-mix.raw
     drive_c\dsound-mix.txt
     ```
   - Example txt:
     ```
     rate=48000 ch=2 bits=32 tag=0xfffe block=8
     ```

3. **One-shot playback for short 6ch effects**
   - Clears `DSBPLAY_LOOPING` in `DSOUND_MixInBuffer()`:
     ```c
     if (dsb->pwfx->nChannels == 6 && dsb->buflen <= 65536)
         dsb->playflags &= ~DSBPLAY_LOOPING;
     ```
   - Goal: let the 85 ms effect finish and release its voice.

4. **Targeted volume boost**
   - Only for short 6ch buffers whose volume is between -15 and -90 dB:
     ```c
     nChannels == 6 && buflen <= 65536 &&
     vol < -1500 && vol > -9000
     ```
   - BGM mute (`-10000`) is not boosted; normal music (`-1070`) is not boosted.

> This is a diagnostics + experimental fix hybrid. It writes files and changes volume; it is not a final production patch.

### `dsound-probe.patch`

- Reports 3D/hardware voice counts as 32 in `DSCAPS`.
- Logs `caps` / `create` / `Play` / `QueryInterface` to:
  ```
  C:\dsound-probe.log
  ```
- Capped at 4000 lines per run, but the file is opened in append mode, so multiple launches accumulate. Delete it before testing.

### `dsound-hwcaps.patch`

Caps only, no logging; for cases where probe logs are not wanted.

---

## Diagnostic findings

### 1. Not 3D, not hardware voices

Every buffer in the probe log:

```
flags=0x000580A0
ctrl3d=0
```

No `CTRL3D`, no `LOCHARDWARE`.  
Creation, `QueryInterface(IDirectSoundBuffer8)`, and `Play` all succeed.

### 2. Effects are 6-channel short buffers

```
tag=0xFFFE (WAVE_FORMAT_EXTENSIBLE)
ch=6, rate=48000, bits=16
buflen=49152 → 4096 frames × 6ch × 2B ≈ 85 ms
```

There is also an 8ch / 102400-byte buffer, likely BGM/ambience streaming.

### 3. Data exists, but volume is very low

Captured raw PCM has real waveform, for example:

```
ch0 rms=8788 max=31000
ch1 rms=8550 max=31052
ch2..5 = 0
```

But the corresponding `vol` is commonly:

```
-2570   about -25.7 dB
-10000  about -100 dB, effectively mute
```

`-10000` is `DSBVOLUME_MIN`; this is why even the first two menu clicks are inaudible in the original build.

### 4. After forcing volume, the first two menu sounds return, the third is still lost

With the mixer forcing short 6ch effects to `1.0`:

- the first two menu key presses are audible;
- the third and later ones are not;
- `dsound-probe.log` shows `Play` called hundreds of times, almost all on two buffer pointers:
  ```
  play buf=1067AE80 flags=0x00000001 ctrl3d=0
  play buf=0AE95BB8 flags=0x00000001 ctrl3d=0
  ```

### 5. Current hypothesis

One of:

1. **ADX has an internal UI voice pool of only 2**, and drops the third event; or
2. **Wine's buffer status / looping semantics** make ADX believe the voices are still busy, so it keeps calling `Play(LOOPING)` and the third event never gets a voice.

This is not confirmed; deeper ADX behaviour analysis is needed.

---

## On-device testing

1. Back up:
   ```
   drive_c\windows\syswow64\dsound.dll
   ```
2. Replace it with the `dsound.dll` from the artifact.
3. The launcher performs runtime verification:
   - boot the original DLL once to create `launcher\startup-verified.dat`;
   - then replace the DLL; a valid cache avoids the full SHA check.
4. Delete old diagnostic files before testing:
   ```
   drive_c\dsound-probe.log
   drive_c\dsound-mix.raw
   drive_c\dsound-mix.txt
   drive_c\dsound-voice-*.raw
   drive_c\dsound-voice-*.txt
   ```
5. Go to the main menu, press keys repeatedly, enter training and kick; observe audio and generated files.

---

## Known limitations

- The third and later short effects may still be lost; the root cause is not fully confirmed.
- `dsound-dump.patch` writes diagnostic files and forces some volume; not suitable for long-term use.
- Forcing 6ch volume can make BGM/ambience louder. The latest version removes the `SetVolume` clamp that overrode BGM mute, but BGM control still needs testing.
- Tested only with FEXTendo 0.4.2 and specific Wine-NX/Box64 versions; other versions may behave differently.
- No guarantee against launcher verification/auto-repair. Always back up before replacing.

---

## Reporting

The most relevant issue tracker is FEXTendo:

- <https://github.com/Ibnuard/pes13_nx/issues/12>

Please attach:

- `dsound-probe.log`
- `dsound-mix.raw` / `dsound-mix.txt`
- `dsound-voice-*.raw` / `.txt`
- this repo's `dsound-dump.patch`
- test steps and expected/actual results

Describe observations rather than conclusions:

> After forcing 6-channel volume to full scale, the first two menu UI sounds are audible, the third and later ones are not. The probe log shows repeated `Play(DSBPLAY_LOOPING)` on only two buffer pointers, suggesting either an ADX-internal voice limit or Wine's buffer status/looping semantics preventing voice reuse.

If it can be reproduced on **vanilla Wine** (not Wine-NX), it can also be reported to Wine Bugzilla under the `dsound` component; otherwise upstream will likely ask for a vanilla Wine reproduction first.

---

## Disclaimer

- Personal research/archive project, provided as-is with no warranty.
- Replacing system DLLs can prevent the game from starting, corrupt saves, or trigger launcher auto-repair.
- Back up `dsound.dll` and your saves.
- Not affiliated with Konami, CRI, FEXTendo, or the Wine project.
