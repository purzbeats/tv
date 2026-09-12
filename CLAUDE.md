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
- **No libm in a per-pixel loop.** `Math.sin/cos/atan2/pow/log` cost ~5-7ns each; at 30000 pixels a single one is 0.2ms and six is most of a frame. Use `fsin`/`fcos`/`fatan2` (below), hoist anything that only depends on `t` out of the loop, precompute anything that only depends on `x`/`y` into a table at load, and collapse a `sin`-into-`rgb()` tail into a `ramp()`. This is the rule the whole file was retuned around; the budget table in **Verifying changes** is how you check it.
- **Nothing allocates inside `draw`.** A `[[...],[...]]` colour table or a scratch `new Float32Array` in a pixel loop is thousands of objects a frame, and GC pause is what actually breaks an ambient display. Hoist to pack scope and reuse.
- **Load-time cost is shared by all 256 scenes.** Keep top-level precompute small; put anything large in `reset()`.

## Architecture

All code lives in one IIFE inside `index.html`, in this order:

1. **Framebuffer.** `W=200, H=150`. `imgA`/`A` are an `ImageData` and a `Uint32Array` view over the same bytes; `B` is a second scene buffer used only during transitions. **Never name a local `A` or `B`** - three packs used to shadow the framebuffers with local palettes, which is silent and would be very hard to debug if someone reached for `A` meaning the framebuffer. Pack palettes are `P`, `PAL`, or a prefixed name. Pixels are packed little-endian ABGR by `rgb(r,g,b)` — blue is the high byte.
2. **Palettes.** `pal()` (cosine gradient), `stops()` (keyed gradient), `hexpal()` (0xRRGGBB list) all return `Uint32Array` lookup tables; most scenes quantize a float into a palette index, which is what produces the banded retro look. Hardware palettes `C64`, `NESP`, `GB`, `ZX`, `CGA`, `AMI` are the real thing — use them for platform-flavoured scenes instead of inventing colours. `BY`/`bay(x,y)` is the 8x8 ordered dither matrix, used both for shading and for the scene transition.
3. **Fast math.** `fsin`/`fcos` read an 8192-entry table (max error 7.7e-4, about 0.2 of an 8-bit colour step) and are ~12x faster than `Math.sin`/`Math.cos`; `fatan2` is a polynomial, ~1.7x faster, error 1.5e-3 rad. At 200x150 with palettised output none of that is visible, so prefer them everywhere in a pixel loop. The table index is truncated with `|0`, which is exact until the argument passes ~1.6e6 radians (~19 days of uptime) and then re-rolls the phase rather than breaking. `ramp(n,lo,hi,fn)` bakes a scalar-to-colour tail into a lookup table - the plasma family all end in "sum three sines, then three more to make an `rgb()`", and `ramp` turns that second half into one array index.
4. **Helpers.** `textMask(str,px)` renders text from the built-in **5x7 bitmap font** (`GLYPH`) into a 1-bit mask — the only text mechanism. `px` is a requested cap height that snaps to a whole-pixel scale (`s = round(px/8)`), so glyphs are always integer multiples of 5x7 and stay crisp. Advance is `5*s + max(1,s-1)` — monospace, which is what keeps the terminal-style screens (BIOSBOOT, BASICLIST, SNESMENU, NESSHOP) column-aligned. At 6px per character a line fits ~32 characters; longer strings clip. `spr(rows)` builds a sprite from ASCII-art strings (`.`/space transparent, digits = palette index) and `blit`/`blit2` draw it. Drawing: `pset/pget/hline/vline/rect/box2/disc/ring/line/tri/addp/fadeBuf/vgrad`. Noise: `hash2/vnoise/fbm`. All take the target buffer first.
5. **Scenes**, grouped into packs, each pack in its own block scope so its shared tables and local helpers stay private:
   - core (32) — the original demo effects: plasma, tunnels, fire, raymarching, fractals, VHS
   - c64 (32) — PETSCII, DYCP, FLD, Kefrens, raster bars, colour clash, boot screens
   - nes (32) — tile/attribute rendering, parallax levels, sprite flicker, game attract modes
   - snes (32) — Mode 7 (a proper focal-length implementation), HDMA, colour math, mosaic
   - amiga (32) — copper, blitter bobs, glenz vectors, cracktro, tracker UI, plus arcade cabinets
   - para (32) — pure multi-layer parallax scrolling scenes
   - frac (17) + auto (15) — escape-time fractals, IFS, L-systems, and cellular automata.
     The shared `escape()` helper renders at half res into 2x2 blocks like everything else
     expensive; it used to be full res, which made it the slowest pack in the file and made
     the escape boundary a one-pixel band that shimmered on a CRT. MANDELBROT, NEWTON,
     ORBITTRAP and JULIAGALLERY hand-roll the same 2x2 pattern. This is the one visible
     change from the optimisation pass: the fractals are chunkier on purpose.
   - d3 (13) + part (9) + misc (10) — wireframe/solid 3D, particle systems, attractors, screen effects
6. **Running order.** Scenes are interleaved round-robin by `tag` so neighbouring parts never come from the same family. Note the interleave keys off the ten *tags*, not the eight authoring packs — `frac`/`auto` and `d3`/`part`/`misc` each rotate independently, so the uneven counts above are deliberate and changing one shifts the whole running order. Adding a scene with a known tag slots it in automatically; an unknown tag is appended as its own family.
7. **Sequencer.** `LEN=15s` per scene, `TRANS=1.3s` Bayer-dither dissolve (a full cycle is ~64 minutes). During a transition the outgoing scene draws into `A`, the incoming into `B`, and pixels are swapped where `bay(x,y) < threshold`. A transition is the only moment two scenes render in one frame, so it is the worst case on slow hardware: `frame()` keeps a smoothed `cost` in ms and, if that is over `BUDGET` (12ms), redraws the incoming scene only every other frame. `B` persists and the dither threshold still advances every frame, so the dissolve stays smooth. On anything desktop-class `cost` sits far below the threshold and this never engages.
8. **Layout + input.** Nothing is drawn over an effect unless the viewer asks: `I` calls `flash()`, which alpha-blends the current effect's name (plus a drop shadow, so it survives bright scenes) over the frame and eases it out over ~3s. Scene changes are deliberately silent — this is an ambient display first. `fit()` computes an integer (or half-step) CSS scale on resize. `window.__demo = { jump, scenes, names }` is exposed purely as a test hook.

   The full input map (single `keydown` handler at the bottom of the file): `space`/`→` next, `←` previous, `1`–`9` jump, `R` random, `P` toggle hold, `I` name flash, `F` fullscreen, `H` show the HUD, click anywhere for next. The HUD (`showHud()`) is a DOM overlay outside the framebuffer that also appears on `mousemove` and self-hides after 2.5s — it is the one exception to "nothing is drawn over an effect", and it never touches `A`/`B`.

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
- **Loop-invariant trig hides well.** `map()` in BLOBJECT computed five `sin`/`cos` of `t` - frame constants - inside a raymarcher called ~360k times a frame, which was 1.8M libm calls to get the same five numbers. Before optimising a scene, check what in its inner loop does not actually depend on the loop variable.
- **Per-cell is not per-pixel.** WORLEY rebuilt the same 80 feature points 30000 times a frame. If a value is indexed by a coarse grid, build the grid once.
- **`hash2` must stay well-mixed.** The original multiply-shift version only ever returned 0.0–0.6, which silently flattened `fbm`, the voxel terrain and every hash-driven tile map. It now uses `Math.imul` mixing; verify distribution if you touch it.

## Verifying changes

There is no test suite. Verification is visual and empirical, and it matters — most effects needed two or three rounds of tuning against real screenshots. Playwright is installed globally but has no browsers downloaded, so drive Edge via `chromium.launch({ channel: 'msedge' })`.

Use the Bash tool for these (the shell is PowerShell by default on this machine; the snippets below are POSIX sh). `$SCRATCH` is the session scratchpad directory — the harnesses are throwaway and should never be committed.

```bash
# syntax check the inline script
sed -n '/^<script>/,/^<\/script>/p' index.html | sed '1d;$d' > "$SCRATCH/chk.js"
node --check "$SCRATCH/chk.js"

NODE_PATH="C:/Users/groov/AppData/Roaming/npm/node_modules" node <harness>.js
```

Harnesses live in the session scratchpad (recreate as needed — they are small):

- **`sheet.js <from> <to> <out.png>`** — jumps to each scene, waits for it to settle, measures fps, grabs `canvas.toDataURL()`, and tiles them into one labelled contact sheet. This is the main tool: **one image verifies 32 effects at a glance.** Reading the sheet is not optional — an effect that runs at 60fps with no errors can still be a wall of noise, an off-screen object, or a solid clipped blob.
- **`fullcheck.js`** — sweeps all 256 scenes for `pageerror`s and sub-50fps frames without screenshotting (~3 min). Run before calling any change done.
- **`pick.js <i> <i> …`** — contact sheet of specific indices with a longer settle time, for scenes that need to accumulate (DLA, SEEDS, LANGTON, attractors).
- **`bench.js <file> <out.json>`** — the one to reach for first. Calls every scene's `draw` directly against a scratch buffer and reports ms/frame, so it measures cost without vsync hiding the headroom. Sorted worst-first with a percent-of-budget column, and it prints the distribution. Takes ~30s for all 256.
- **`throttle.js <file> <rate> <scene> …`** — real rAF frame rate with the CPU throttled via CDP, as a stand-in for Pi-class silicon. `6x` is roughly Pi 4/5; everything holds 60fps there. `12x` is where the six heaviest scenes start to drop.

**Budget.** 16.7ms is a frame. Aim for **under 1ms per scene** on a desktop: that leaves ~6x headroom for a Pi, and a transition can draw two scenes in one frame. As of the optimisation pass the mean is 0.12ms, 243 of 256 scenes are under 0.5ms, and the worst (LATTICE) is 2.2ms.

Edit `index.html` with targeted Python string replacements or the Edit tool rather than rewriting it; the tuned constants scattered through 256 effects are the result of iteration and are easy to lose.

`docs/*.png` are checked-in contact sheets — one per pack, linked from `README.md`. If a change visibly alters a pack, regenerate that pack's sheet with `sheet.js` and commit it alongside. `preview.png` is the README hero shot.

## Deployment

`vercel.json` serves `index.html` with `max-age=0, must-revalidate` so the deployed copy is never stale. There is nothing to build — Vercel publishes the repo root as static files.

## Other agent config

A Codex config exists at `~/.codex/config.toml`. If you want its MCP servers, commands, or instructions brought over to Claude Code, reply `/import` to see what's importable (or run `claude import` from a terminal if the command isn't available here).
