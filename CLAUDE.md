# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

A single-file demoscene loop (`index.html`) written to be fullscreened in a browser on a CRT TV at roughly 400x300. **256 effects**, everything software-rendered pixel by pixel into a 200x150 buffer and blitted at an integer scale with `image-rendering: pixelated`. No build step, no package manager, no dependencies, no network — the file is opened directly over `file://`.

## Hard constraints

These are the design rules the whole file is built around; breaking one breaks the point of the project.

- **Stay single-file and dependency-free.** No imports, no CDN, no fetch. The user drags `index.html` onto the target machine.
- **Never introduce smoothing.** No `imageSmoothingEnabled`, no CSS transforms with interpolation, no fractional canvas scaling beyond the deliberate half-step in `fit()`. Chunky square pixels are the aesthetic.
- **Every scene must hold 60fps** at 200x150. Expensive effects (raymarching, fractals, cellular automata) render at half resolution (100x75) into 2x2 pixel blocks — follow that pattern rather than dropping the frame rate.
- **Watch spatial frequency.** Detail finer than ~3px reads as shimmering noise at this resolution and looks terrible on a CRT. Many effects were retuned specifically for this (Kefrens bar frequency, moiré ring spacing, Mode 7 fog, grid-line anti-moiré suppression, Menger subdivision — the sponge was eventually replaced by a lattice because it could not be made legible). When something looks like static, the fix is coarser features, not more detail.
- **Load-time cost is shared by all 256 scenes.** Keep top-level precompute small; put anything large in `reset()`.

## Architecture

All code lives in one IIFE inside `index.html`, in this order:

1. **Framebuffer.** `W=200, H=150`. `imgA`/`A` are an `ImageData` and a `Uint32Array` view over the same bytes; `B` is a second scene buffer used only during transitions. Pixels are packed little-endian ABGR by `rgb(r,g,b)` — blue is the high byte.
2. **Palettes.** `pal()` (cosine gradient), `stops()` (keyed gradient), `hexpal()` (0xRRGGBB list) all return `Uint32Array` lookup tables; most scenes quantize a float into a palette index, which is what produces the banded retro look. Hardware palettes `C64`, `NESP`, `GB`, `ZX`, `CGA`, `AMI` are the real thing — use them for platform-flavoured scenes instead of inventing colours. `BY`/`bay(x,y)` is the 8x8 ordered dither matrix, used both for shading and for the scene transition.
3. **Helpers.** `textMask(str,px)` renders text from the built-in **5x7 bitmap font** (`GLYPH`) into a 1-bit mask — the only text mechanism. `px` is a requested cap height that snaps to a whole-pixel scale (`s = round(px/8)`), so glyphs are always integer multiples of 5x7 and stay crisp. Advance is `5*s + max(1,s-1)` — monospace, which is what keeps the terminal-style screens (BIOSBOOT, BASICLIST, SNESMENU, NESSHOP) column-aligned. At 6px per character a line fits ~32 characters; longer strings clip. `spr(rows)` builds a sprite from ASCII-art strings (`.`/space transparent, digits = palette index) and `blit`/`blit2` draw it. Drawing: `pset/pget/hline/vline/rect/box2/disc/ring/line/tri/addp/fadeBuf/vgrad`. Noise: `hash2/vnoise/fbm`. All take the target buffer first.
4. **Scenes**, grouped into packs, each pack in its own block scope so its shared tables and local helpers stay private:
   - core (32) — the original demo effects: plasma, tunnels, fire, raymarching, fractals, VHS
   - c64 (32) — PETSCII, DYCP, FLD, Kefrens, raster bars, colour clash, boot screens
   - nes (32) — tile/attribute rendering, parallax levels, sprite flicker, game attract modes
   - snes (32) — Mode 7 (a proper focal-length implementation), HDMA, colour math, mosaic
   - amiga (32) — copper, blitter bobs, glenz vectors, cracktro, tracker UI, plus arcade cabinets
   - para (32) — pure multi-layer parallax scrolling scenes
   - frac + auto (32) — escape-time fractals, IFS, L-systems, and cellular automata
   - d3 + part + misc (32) — wireframe/solid 3D, particle systems, attractors, screen effects
5. **Running order.** Scenes are interleaved round-robin by `tag` so neighbouring parts never come from the same family. Adding a scene with a known tag slots it in automatically; an unknown tag is appended as its own family.
6. **Sequencer.** `LEN=15s` per scene, `TRANS=1.3s` Bayer-dither dissolve (a full cycle is ~64 minutes). During a transition the outgoing scene draws into `A`, the incoming into `B`, and pixels are swapped where `bay(x,y) < threshold`.
7. **Layout + input.** Nothing is drawn over an effect unless the viewer asks: `I` calls `flash()`, which alpha-blends the current effect's name (plus a drop shadow, so it survives bright scenes) over the frame and eases it out over ~3s. Scene changes are deliberately silent — this is an ambient display first. `fit()` computes an integer (or half-step) CSS scale on resize. `window.__demo = { jump, scenes, names }` is exposed purely as a test hook.

### The scene contract

```js
scenes.push({
  name: 'NAME',                 // shown in the corner flash; must be unique
  tag: 'c64',                   // family, drives the running order
  reset(){ ... },               // optional; called when the scene becomes current or incoming
  draw(T, t, dt){ ... }         // T = target buffer, t = seconds since load, dt = frame delta
});
```

`draw` must write **every** pixel of `T` it cares about, because `T` alternates between `A` and `B` and arrives holding another scene's frame. A scene that leaves pixels untouched will show fragments of its neighbour — this is exactly how the WATER scene's rainbow border bug appeared.

Scenes needing frame-to-frame persistence (FEEDBACK, VECTORS, SHADEBOBS, ATTRACTOR, SCOPE, LORENZ, SIERPINSKI, …) **must own a private accumulation buffer** and `T.set(acc)` at the end, for the same reason. HYPERSPACE, FIREWORKS and MATRIX are deliberate exceptions: they fade `T` in place, which is harmless because `B` is cleared to black when a transition starts.

`reset()` must be idempotent and cheap; it fires again every time the scene comes back around. Scenes that build state lazily use the `if(!x)this.reset();` guard at the top of `draw`.

### Gotchas that have already bitten

- **`|` binds looser than arithmetic.** `(H*0.7)|0-(y-4)` parses as `(H*0.7) | (0-(y-4))`. Parenthesize every `|0` used inside an expression.
- **Fractional array indices silently do nothing.** `T[y*W+x] = c` with float `x`/`y` writes nowhere. Use `pset` or coerce with `|0` — an inline `add()` helper in COLORMATH lost its output this way.
- **Text was previously rasterized from a browser font and thresholded at alpha > 110**, which ate the thin stems of small glyphs and turned labels to mush. It is now a hand-authored bitmap font; don't reintroduce canvas text rendering.
- **`hash2` must stay well-mixed.** The original multiply-shift version only ever returned 0.0–0.6, which silently flattened `fbm`, the voxel terrain and every hash-driven tile map. It now uses `Math.imul` mixing; verify distribution if you touch it.

## Verifying changes

There is no test suite. Verification is visual and empirical, and it matters — most effects needed two or three rounds of tuning against real screenshots. Playwright is installed globally but has no browsers downloaded, so drive Edge via `chromium.launch({ channel: 'msedge' })`.

```bash
# syntax check the inline script
sed -n '/^<script>/,/^<\/script>/p' index.html | sed '1d;$d' > "$SCRATCH/chk.js"
node --check "$SCRATCH/chk.js"

NODE_PATH="C:/Users/groov/AppData/Roaming/npm/node_modules" node <harness>.js
```

Three harnesses live in the session scratchpad (recreate as needed — they are small):

- **`sheet.js <from> <to> <out.png>`** — jumps to each scene, waits for it to settle, measures fps, grabs `canvas.toDataURL()`, and tiles them into one labelled contact sheet. This is the main tool: **one image verifies 32 effects at a glance.** Reading the sheet is not optional — an effect that runs at 60fps with no errors can still be a wall of noise, an off-screen object, or a solid clipped blob.
- **`fullcheck.js`** — sweeps all 256 scenes for `pageerror`s and sub-50fps frames without screenshotting (~3 min). Run before calling any change done.
- **`pick.js <i> <i> …`** — contact sheet of specific indices with a longer settle time, for scenes that need to accumulate (DLA, SEEDS, LANGTON, attractors).

Edit `index.html` with targeted Python string replacements or the Edit tool rather than rewriting it; the tuned constants scattered through 256 effects are the result of iteration and are easy to lose.

## Other agent config

A Codex config exists at `~/.codex/config.toml`. If you want its MCP servers, commands, or instructions brought over to Claude Code, reply `/import` to see what's importable (or run `claude import` from a terminal if the command isn't available here).
