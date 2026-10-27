# AGENTS.md

Instructions for AI coding agents working in this repository.

## What this is

Demoscene-style productions with an Ubuntu / Canonical look. Everything runs
from a single self-contained `index.html` — no build step, no dependencies,
WebGL 1.0 / GLSL ES 1.00 so it runs anywhere, including software rasterizers.

Current production: **ubuntu × audio** — an audio-reactive shader demo
(plasma, tunnel, raymarched logo, metaballs, wireframe vectors, fire) with a
built-in WebAudio chiptune, microphone and file input. No scroller/text overlay
on the show — it was removed on purpose; keep the canvas text-free.

## Rules

- Keep `index.html` one file: inline the shader, the synth and the UI. No
  build steps, no npm dependencies, no external assets.
- Palette is the Canonical core: orange `#E95420`, aubergines
  `#77216F`/`#5E2750`/`#2C001E`, warm grey `#AEA79F`, white `#F7F7F7`
  (constants `C_*` in the shader).
- Never import the real Ubuntu logo asset. The mark is a hand-made procedural
  SDF approximation (`sdLogo`); keep the trademark disclaimer in the README.
- Apache-2.0 (see LICENSE). Commits: single line, `<area>: lowercase summary`.

## How it works

- One full-screen fragment shader; the timeline loops through 6 scenes of
  `SEG = 16` s each (plasma, tunnel, raymarched logo, metaballs, wireframe
  vectors, fire), crossfading over the last 2.5 s. `uTime` drives everything.
- **Debug stills: `index.html?t=<seconds>`** freezes the timeline — the way to
  check a specific scene deterministically (e.g. `?t=36` for the raymarch scene).
- Audio: `AnalyserNode` → 256-bin spectrum → 256×1 LUMINANCE texture (`uAudio`)
  → sampled per pixel; smoothed `bass/mid/high` + kick detector feed `uLevels`.
  With no audio playing, a synthetic groove feeds the same uniforms.
- Microphone level: `micSrc → micGain → analyser`, a ±20 dB slider in the HUD
  (mic mode only) plus an adaptive mode (default, `micAutoOn`) that chases a
  0.32 loudness target at ≤8 dB/s, gated above the noise floor so silence
  never cranks the gain. Level and auto flag persist in localStorage
  (`uaxdb`, `uaxauto`); Chrome's `autoGainControl` is disabled in getUserMedia.
  `window.__demoLevels` exposes micDb / auto in slots 6/7.
- The Ubuntu typeface is embedded as base64 `@font-face` (UFL 1.0,
  LICENSE-UbuntuFont) for the DOM UI (overlay title, buttons, HUD).

## Gotchas — read before touching the shader

- **`gl_FragCoord.y` is 0 at the screen BOTTOM.** Screen-space overlays are
  anchored from the bottom: the spectrum bars live at `suv.y < 0.16`.
- Smoothstep edges must increase (`edge0 < edge1`); GLSL leaves the reverse
  undefined — write `1.0 - smoothstep(a, b, x)` instead.

## Verifying changes headlessly

- Use `chrome-headless-shell` (Chrome for Testing zip) + `puppeteer-core` on
  SwiftShader: `--no-sandbox --use-gl=angle --use-angle=swiftshader
  --enable-unsafe-swiftshader`, and **`defaultViewport: null`** so
  `--force-device-scale-factor` actually applies (puppeteer otherwise pins
  DPR=1 and HiDPI is never tested). Test DPR 1 and 2 and a short-wide window
  (~1394×269) — that is a real viewing layout.
- Verify visually with one still per scene: `?t=6`, `?t=22`, `?t=38`, `?t=54`,
  `?t=70`, `?t=86` — assert content, palette and brightness profile per
  scene (thin bright lines average away in downscaled analysis — judge peaks,
  not means). A partially-black desktop capture is usually a compositor/GPU
  glitch on the capturing machine, not a shader bug — reproduce headlessly
  before "fixing" one.
- **Fonts:** the Ubuntu face is embedded for the DOM UI, but headless boxes
  may still have zero *system* fonts — on such machines even the embedded
  webfont fails to load (faces stuck in `status: "error"`). Drop a TTF into
  `~/.local/share/fonts` (e.g. DejaVu from the gnome snap) before judging
  DOM text rendering.
- Snap-confined Chromium cannot run headless on this machine (D-Bus AppArmor
  fatal); that is why the standalone `chrome-headless-shell` zip is used.
- A tmp cleaner periodically wipes `/tmp` — a long pause costs the whole
  toolchain. Keep chrome-headless-shell, puppeteer-core and the test scripts
  in `~/demoscene-tools` (the demo server: `python3 -m http.server 8019` from
  the demo directory). Mic behaviour is testable headlessly with
  `--use-fake-device-for-media-stream --use-fake-ui-for-media-stream`.
