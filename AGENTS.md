# AGENTS.md

Instructions for AI coding agents working in this repository.

## What this is

Demoscene-style productions with an Ubuntu / Canonical look. Everything runs
from a single self-contained `index.html` — no build step, no dependencies,
WebGL 1.0 / GLSL ES 1.00 so it runs anywhere, including software rasterizers.

Current production: **ubuntu × audio** — an audio-reactive shader demo
(plasma → tunnel → raymarched logo) with a built-in WebAudio chiptune,
microphone and file input.

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

- One full-screen fragment shader; the timeline loops through 3 scenes of
  `SEG = 16` s each, crossfading over the last 2.5 s. `uTime` drives everything.
- **Debug stills: `index.html?t=<seconds>`** freezes the timeline — the way to
  check a specific scene deterministically (e.g. `?t=36` for the raymarch scene).
- Audio: `AnalyserNode` → 256-bin spectrum → 256×1 LUMINANCE texture (`uAudio`)
  → sampled per pixel; smoothed `bass/mid/high` + kick detector feed `uLevels`.
  With no audio playing, a synthetic groove feeds the same uniforms.
- Greetings scroller: `MESSAGE` baked once into a 2D canvas strip (height 128,
  width = POT ladder up to `MAX_TEXTURE_SIZE`, font shrunk to fit), uploaded as
  `uText` with `REPEAT` wrap. The shader samples it in a band and `uTextSz`
  carries the strip dimensions.

## Gotchas — read before touching the shader or scroller

- **`gl_FragCoord.y` is 0 at the screen BOTTOM.** All screen-space overlays are
  anchored from the bottom: spectrum bars at `suv.y < 0.16`, scroller band at
  `suv.y` 0.175–0.305. From the top of a H×W screenshot the band is at rows
  `0.695·H … 0.825·H`. Measure there, not near the top.
- **`UNPACK_FLIP_Y_WEBGL` must be `true` for the scroller upload.** Band-bottom
  samples V=0; only the flipped upload puts the canvas bottom row there. Remove
  the flip and the glyphs render upside down.
- **Texture-U offsets are screen-width units.** Express slant, scroll speed and
  shadow offset in texture pixels divided by `uTextSz.x` so they survive strip
  and window changes. (A raw `0.28` slant once produced ~75° shear.)
- **Match horizontal sampling scale to the vertical one** or glyphs compress:
  `uspan = uRes.x*uTextSz.y/(uTextSz.x*0.13*uRes.y)`.
- Size the scroller font to fill the band (target ~88 px in the 128 px canvas)
  and lengthen the strip, never the reverse — small text lost in the band is a
  bug, not a style.

## Verifying changes headlessly

- Use `chrome-headless-shell` (Chrome for Testing zip) + `puppeteer-core` on
  SwiftShader: `--no-sandbox --use-gl=angle --use-angle=swiftshader
  --enable-unsafe-swiftshader`, and **`defaultViewport: null`** so
  `--force-device-scale-factor` actually applies (puppeteer otherwise pins
  DPR=1 and HiDPI is never tested). Test DPR 1 and 2 and a short-wide window
  (~1394×269) — that is a real viewing layout.
- **Headless boxes may have zero fonts installed.** `measureText` then returns
  0 and `fillText` draws *nothing* — the scroller is silently invisible and
  brightness-coverage checks still pass. Install a TTF (e.g. DejaVu from the
  gnome snap) into `~/.local/share/fonts` first, then assert the strip has
  opaque pixels *and* read the rendered band.
- Verify visually at `?t=0`, `?t=16`, `?t=36`: upright glyphs, subtle slant
  (~9°), 1:1 aspect, text filling the band. A partially-black desktop capture
  is usually a compositor/GPU glitch on the capturing machine, not a shader
  bug — reproduce headlessly before "fixing" one.
- Snap-confined Chromium cannot run headless on this machine (D-Bus AppArmor
  fatal); that is why the standalone `chrome-headless-shell` zip is used.
